# Background worker xử lý job dài đang consume message từ queue

**Hệ thống:** Một cụm worker consume message từ hàng đợi để xử lý các job chạy lâu (export file dữ liệu lớn, gửi email hàng loạt, xử lý batch report), mỗi job có thể mất từ vài chục giây tới vài phút để hoàn tất.

**Vai trò của flow:** Khi worker bị shutdown (deploy, autoscale scale-in), phải đảm bảo message đang xử lý dở không bị mất và không bị ack sai thời điểm — ack quá sớm làm mất dữ liệu nếu worker chết giữa chừng, ack quá muộn hoặc không ack làm message bị xử lý lặp lại không cần thiết.

**Yêu cầu cụ thể:**
- Khi nhận tín hiệu shutdown, worker phải ngay lập tức ngừng nhận message mới từ queue (ngừng poll/prefetch) nhưng vẫn tiếp tục xử lý cho tới khi hoàn tất các message đã nhận trước đó, tránh tình huống nhận thêm việc mới ngay trước lúc tắt rồi phải bỏ dở.
- Message chỉ được ack (xác nhận đã xử lý xong, xóa khỏi queue) sau khi job đã hoàn tất toàn bộ và kết quả đã được ghi bền vững — nếu worker bị kill cứng giữa lúc xử lý (hết grace period), message chưa ack phải tự động được đưa trở lại queue để worker khác xử lý lại, không được để mất.
- Với các job có thể để lại tác động một phần khi bị ngắt giữa chừng (ví dụ export file đã ghi được nửa file, gửi email hàng loạt đã gửi được một số người trong danh sách), phải thiết kế idempotent hoặc có checkpoint để lần xử lý lại sau không lặp lại phần đã hoàn tất (không gửi trùng email cho người đã nhận, không tạo file export bị lỗi nửa vời).
- Định nghĩa grace period đủ dài để job dài nhất trong hệ thống có cơ hội hoàn tất bình thường (không cắt ngang job đang chạy gần xong), nhưng vẫn có ngưỡng tối đa hợp lý để không làm chậm quá trình deploy/scale toàn cụm — cân nhắc cho phép job vượt quá một độ dài nhất định tự chủ động lưu checkpoint giữa chừng thay vì cố chờ nó chạy xong trong grace period.
- Xử lý trường hợp toàn bộ worker trong cụm đều nhận tín hiệu shutdown gần như đồng thời (ví dụ rolling deploy toàn cụm) — phải đảm bảo còn ít nhất một số worker tiếp tục nhận message mới trong lúc số còn lại đang drain, tránh để hàng đợi bị dồn ứ không ai xử lý trong lúc chuyển giao.

---

# Base ERD — Background worker consume queue trước khi có graceful shutdown

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** worker biết xử lý shutdown an toàn. Đề bài nói tới cụm worker consume message từ hàng đợi để xử lý job dài — nên base chỉ cần đủ: hàng đợi (QUEUE), job/message trong hàng đợi (JOB) và worker đang xử lý (WORKER). Chưa có checkpoint, chưa có visibility timeout theo dõi rõ ràng, chưa có trạng thái draining, chưa có kế hoạch phối hợp rolling shutdown toàn cụm — tất cả những thứ đó là phần enhance.

```mermaid
erDiagram
    QUEUE ||--o{ JOB : contains
    WORKER ||--o{ JOB : processes

    QUEUE {
        string id PK
        string name
    }
    JOB {
        string id PK
        string queue_id FK
        string type "export | bulk_email | batch_report"
        string payload
        string status "pending | processing | done | failed"
        datetime received_at
    }
    WORKER {
        string id PK
        string status "running"
    }
```

---

# Base sequence — Consume job (ack sớm, không chống mất dữ liệu khi crash)

Đây là **base**, flow "Worker lấy job từ queue và xử lý" ở trạng thái hiện tại — worker ack message ngay sau khi nhận (trước khi xử lý xong) để tránh bị nhận trùng, và không có cơ chế nào xử lý riêng khi có tín hiệu shutdown giữa lúc đang chạy job dài. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu của đề bài xoay quanh việc sửa đúng thời điểm ack và cách worker phản ứng khi bị tắt.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W as Worker
    participant Store as Result Store

    Queue->>W: Deliver message (job=export_report_123)
    W->>Queue: ACK ngay lập tức (để tránh bị deliver lại)
    Note over Queue,W: Từ giờ queue coi job này đã xử lý xong, không giữ lại bản sao nào nữa
    W->>W: Bắt đầu xử lý export file (job mất 3 phút)

    Note over W: Giữa chừng, deploy mới trigger, hạ tầng kill cứng process worker (không có xử lý shutdown riêng)
    W--xW: Process bị kill, file export mới ghi được 40%

    Note over Queue,Store: Message đã bị ACK từ đầu nên queue không còn bản sao để giao lại cho worker khác, job coi như mất, file export dở dang không ai biết để dọn hoặc làm lại
```

---

# Enhance ERD — Thêm checkpoint, visibility timeout, trạng thái draining, kế hoạch drain cụm

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi:
- `JOB` có thêm cột `visibility_timeout_at` (job coi như "đang được ai đó xử lý" tới thời điểm này, hết hạn mà chưa ack thì tự động quay lại queue) và `checkpoint` (lưu tiến độ xử lý dở, ví dụ số dòng file đã ghi/số người đã gửi email) — đáp ứng yêu cầu 2 và yêu cầu 3.
- `WORKER` có thêm trạng thái `draining` và các cột `grace_period_seconds`, `shutdown_deadline_at` — đáp ứng yêu cầu 1 và yêu cầu 4 (định nghĩa grace period rõ ràng).
- Entity mới `CLUSTER_DRAIN_PLAN` — theo dõi 1 đợt rolling deploy/scale toàn cụm, đảm bảo luôn còn tối thiểu 1 số worker ở trạng thái active nhận message mới trong lúc số còn lại đang drain — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    QUEUE ||--o{ JOB : contains
    WORKER ||--o{ JOB : processes
    CLUSTER_DRAIN_PLAN ||--o{ WORKER : coordinates

    QUEUE {
        string id PK
        string name
    }
    JOB {
        string id PK
        string queue_id FK
        string type "export | bulk_email | batch_report"
        string payload
        string status "pending | processing | done | failed"
        datetime received_at
        datetime visibility_timeout_at
        string checkpoint "tien do xu ly do dang, vd so nguoi da gui email"
    }
    WORKER {
        string id PK
        string status "active | draining | stopped"
        int grace_period_seconds
        datetime shutdown_deadline_at
    }
    CLUSTER_DRAIN_PLAN {
        string id PK
        string deploy_id
        int total_workers
        int workers_draining
        int workers_active
        int min_active_workers "luon giu toi thieu bao nhieu worker con nhan message moi"
    }
```

---

# Enhance sequence — Consume job (ngừng prefetch ngay, ack sau khi xong và ghi bền vững)

Đây là **enhance** của flow `consume-job` đã có ở base, đáp ứng **yêu cầu 1 và 2** của đề bài. So với base: khi nhận tín hiệu shutdown, worker ngừng ngay việc poll/prefetch message mới nhưng tiếp tục hoàn tất job đang xử lý dở, và chỉ ACK sau khi job đã hoàn tất toàn bộ và kết quả đã ghi bền vững — không còn ack ngay sau khi nhận message như base.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W as Worker
    participant Store as Result Store

    Queue->>W: Deliver message (job=export_report_123, visibility_timeout=5 phút)
    W->>W: Bắt đầu xử lý export file (không ACK vội)

    Note over W: Giữa lúc đang xử lý, nhận tín hiệu shutdown (SIGTERM)
    W->>Queue: Ngừng poll/prefetch message mới ngay lập tức
    W->>W: Tiếp tục xử lý job đang dở tới khi hoàn tất (không nhận thêm job mới)

    W->>Store: Ghi file export hoàn chỉnh (bền vững)
    Store-->>W: OK
    W->>Queue: ACK job=export_report_123 (chỉ ack sau khi ghi xong)
    Queue-->>W: Xác nhận xoá message khỏi queue

    W->>W: Không còn job nào đang xử lý dở, worker tắt hẳn
```

---

# Enhance sequence — Hết grace period, bị kill cứng, message tự động quay lại queue

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng phần còn lại của **yêu cầu 2**: nếu worker bị kill cứng giữa lúc xử lý (hết grace period mà job chưa xong), message chưa được ack phải tự động được đưa trở lại queue nhờ `visibility_timeout_at` hết hạn, để worker khác xử lý lại, không được để mất.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W1 as Worker A (đang xử lý)
    participant Orchestrator as Deploy Orchestrator
    participant W2 as Worker B (worker khác trong cụm)

    Queue->>W1: Deliver message (job=bulk_email_88, visibility_timeout=5 phút)
    W1->>W1: Bắt đầu xử lý (nhận tín hiệu shutdown, đang trong grace period)
    Note over W1: Job vẫn chưa xong khi hết grace_period_seconds đã định nghĩa

    Orchestrator->>W1: Hết grace period, kill cứng process
    W1--xW1: Process bị chấm dứt, message chưa được ACK

    Note over Queue: visibility_timeout_at của message này tới hạn (worker A không còn heartbeat/extend), queue tự động coi message là chưa xử lý xong
    Queue->>Queue: Đưa message trở lại trạng thái pending, sẵn sàng giao lại

    Queue->>W2: Deliver lại message (job=bulk_email_88)
    W2->>W2: Xử lý lại job từ đầu (hoặc từ checkpoint nếu có, xem flow checkpoint-idempotent-job)
```

---

# Enhance sequence — Checkpoint cho job dài, tránh gửi trùng khi xử lý lại

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 3 và yêu cầu 4** của đề bài: job gửi email hàng loạt có thể để lại tác động một phần khi bị ngắt giữa chừng, nên phải lưu checkpoint (đã gửi tới người thứ mấy) để lần xử lý lại sau không gửi trùng. Vì job này dài hơn grace period tối đa cho phép, worker chủ động lưu checkpoint giữa chừng thay vì cố chờ chạy xong trong grace period.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W1 as Worker A
    participant Store as Checkpoint Store
    participant W2 as Worker B (xử lý lại sau)

    Queue->>W1: Deliver message (job=bulk_email_88, danh sách 10000 người nhận)
    W1->>W1: Bắt đầu gửi email lần lượt

    loop Mỗi 500 email đã gửi thành công
        W1->>Store: UPDATE checkpoint = {sent_up_to_index: 500}
    end

    Note over W1: Job này ước tính mất 12 phút, dài hơn grace_period_seconds tối đa (30 giây) nên không cố chờ xong trong grace period
    W1->>W1: Nhận tín hiệu shutdown khi đã gửi tới người thứ 3200
    W1->>Store: UPDATE checkpoint = {sent_up_to_index: 3200} (lưu ngay trước khi hết grace period)
    W1->>Queue: Không ACK (để message quay lại queue cho worker khác tiếp tục)
    W1->>W1: Hết grace period, tắt hẳn

    Queue->>W2: Deliver lại message (job=bulk_email_88)
    W2->>Store: SELECT checkpoint WHERE job_id=bulk_email_88
    Store-->>W2: {sent_up_to_index: 3200}
    W2->>W2: Tiếp tục gửi từ người thứ 3201 trở đi, không gửi trùng cho 3200 người đã nhận
    W2->>Store: Ghi checkpoint cuối cùng khi gửi xong toàn bộ 10000 người
    W2->>Queue: ACK job=bulk_email_88
```

---

# Enhance sequence — Toàn cụm nhận tín hiệu shutdown gần đồng thời, vẫn giữ worker tiếp nhận việc mới

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: khi rolling deploy khiến toàn bộ worker trong cụm nhận tín hiệu shutdown gần như đồng thời, `CLUSTER_DRAIN_PLAN` đảm bảo luôn còn tối thiểu một số worker tiếp tục nhận message mới trong lúc số còn lại đang drain, tránh hàng đợi bị dồn ứ không ai xử lý.

```mermaid
sequenceDiagram
    participant Orchestrator as Deploy Orchestrator
    participant Plan as CLUSTER_DRAIN_PLAN
    participant W1 as Worker 1
    participant W2 as Worker 2
    participant W3 as Worker 3
    participant Queue as Message Queue

    Orchestrator->>Plan: Bắt đầu rolling deploy cụm (total_workers=3, min_active_workers=1)
    Orchestrator->>W1: Gửi tín hiệu shutdown
    Orchestrator->>W2: Gửi tín hiệu shutdown
    Orchestrator->>W3: Gửi tín hiệu shutdown

    W1->>Plan: Xin phép chuyển sang draining
    Plan-->>W1: OK (workers_active sau khi trừ W1 vẫn >= min_active_workers)
    W1->>W1: status=draining, ngừng nhận message mới, xử lý nốt job dở

    W2->>Plan: Xin phép chuyển sang draining
    Plan-->>W2: OK (workers_active vẫn còn W3 >= min_active_workers)
    W2->>W2: status=draining, ngừng nhận message mới

    W3->>Plan: Xin phép chuyển sang draining
    Plan-->>W3: Từ chối tạm thời (nếu W3 drain ngay, workers_active=0 < min_active_workers)
    W3->>W3: Vẫn giữ status=active, tiếp tục poll và xử lý message mới từ Queue

    Note over W1,W2: W1 và W2 hoàn tất drain lần lượt, mỗi worker khi restart xong báo lại Plan để cập nhật workers_active
    W1->>Plan: Đã restart xong, workers_active += 1
    Plan->>W3: Cho phép W3 bắt đầu drain (đã có worker khác thay thế)
    W3->>W3: status=draining, ngừng nhận message mới, xử lý nốt job dở rồi restart
```
