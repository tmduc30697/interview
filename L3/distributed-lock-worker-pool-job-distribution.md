# Phân việc cho worker pool xử lý hàng đợi công việc lớn

**Hệ thống:** Nhiều worker instance chạy song song để xử lý hàng loạt job từ một hàng đợi công việc lớn (ví dụ xử lý file, gửi batch email), cần chia việc sao cho không trùng lặp và không bỏ sót.

**Vai trò của flow:** Distributed lock/leader election theo từng job (hoặc theo phân vùng job) đảm bảo mỗi job chỉ được đúng một worker nhận và xử lý tại một thời điểm, đồng thời không bỏ sót job khi worker chết giữa chừng.

**Yêu cầu cụ thể:**
- Worker phải "claim" một job bằng lock/lease có TTL trước khi bắt đầu xử lý; nếu worker chết giữa chừng (crash, bị kill do autoscale down) mà không release lock, TTL hết hạn phải tự đưa job đó về trạng thái "khả dụng" để worker khác nhận lại — không để job kẹt vĩnh viễn ở trạng thái "đang xử lý" bởi worker đã chết.
- Hai worker cùng lúc thấy job "chưa ai nhận" và cùng cố gắng claim (race condition khi liệt kê job và acquire lock không atomic với nhau) phải được giải quyết sao cho chỉ đúng một worker thắng, worker thua phải nhận biết ngay để không tiếp tục xử lý song song, tránh xử lý trùng và ghi kết quả hai lần.
- Nếu job cần thời gian xử lý lâu hơn TTL ước tính ban đầu (ví dụ file lớn bất thường), worker đang xử lý phải renew lease định kỳ; nếu renew thất bại (mất kết nối coordinator) trong khi job vẫn đang chạy thật, cần cơ chế idempotent hoặc kiểm tra trạng thái trước khi apply kết quả để chịu được việc worker khác nhận lại job đó khi lock hết hạn.
- Khi một worker bị autoscale terminate có báo trước (graceful shutdown signal), phải chủ động release lock/trả lại job đang giữ ngay lập tức thay vì chờ TTL hết hạn, để giảm độ trễ job bị "treo" chờ timeout tự nhiên.
- Cần cơ chế định kỳ quét các job ở trạng thái "đang xử lý" quá lâu so với TTL kỳ vọng (dead job detection) để phát hiện sớm rò rỉ lock hoặc lỗi hệ thống, tránh dựa hoàn toàn vào TTL tự nhiên mà không có giám sát chủ động.

---

# ERD - Base (trước khi có distributed lock/lease cho job)

Đây là trạng thái **base**: có một hàng đợi công việc lớn (`JOB`) và nhiều `WORKER` chạy song song để xử lý, nhưng việc "ai nhận job nào" chỉ dựa vào cập nhật trạng thái đơn giản, chưa có lock/lease với TTL, chưa có cơ chế phát hiện worker chết. Đây là nền tối thiểu để enhance thêm cơ chế claim bằng lock có TTL, atomic claim, renew, graceful shutdown và dead job detection.

```mermaid
erDiagram
    WORKER ||--o{ JOB : "đang xử lý (nếu có)"

    WORKER {
        string worker_id PK
        string status
        datetime started_at
    }
    JOB {
        string job_id PK
        string job_type
        string payload_ref
        string status
        string assigned_worker_id FK
        datetime created_at
        datetime updated_at
    }
```

---

# Sequence Diagram - Base: claim-and-process-job

Đây là flow **base**: worker định kỳ poll hàng đợi, thấy job đang ở trạng thái "AVAILABLE" thì tự cập nhật thẳng thành "PROCESSING" rồi xử lý, không qua bất kỳ lock/lease nào. Flow này là nền để so sánh với enhance, nơi bước "claim" sẽ được thay bằng thao tác atomic có TTL để tránh 2 worker cùng nhận 1 job.

```mermaid
sequenceDiagram
    participant W as Worker
    participant Q as Job Queue/DB

    W->>Q: Poll danh sách job status=AVAILABLE
    Q-->>W: Trả về job J1
    W->>Q: Update job J1 status=PROCESSING, assigned_worker_id=W
    Q-->>W: OK
    W->>W: Xử lý job J1 (đọc file, gửi email...)
    W->>Q: Update job J1 status=DONE
    Q-->>W: OK
```

---

# Sequence Diagram - Base: worker-crash-job-stuck

Đây là flow **base** minh hoạ vấn đề chính mà đề bài cần giải quyết: khi worker đang xử lý job mà crash (hoặc bị autoscale kill) giữa chừng, job vẫn nằm ở trạng thái "PROCESSING" mãi mãi vì không có TTL/lease nào theo dõi, không worker nào khác dám nhận lại, job bị kẹt vĩnh viễn. Đây là lý do enhance cần TTL tự động trả job về "AVAILABLE" và cơ chế dead job detection.

```mermaid
sequenceDiagram
    participant W as Worker (sắp crash)
    participant Q as Job Queue/DB
    participant W2 as Worker khác

    W->>Q: Update job J1 status=PROCESSING, assigned_worker_id=W
    Q-->>W: OK
    Note over W: Worker bị crash/bị autoscale kill giữa lúc xử lý
    W2->>Q: Poll danh sách job status=AVAILABLE
    Q-->>W2: Không thấy J1 (vẫn đang ở PROCESSING)
    Note over Q: Không có TTL, không ai phát hiện worker W đã chết, J1 kẹt vĩnh viễn ở PROCESSING
```

---

# ERD - Enhance (claim bằng lock/lease có TTL cho job)

Đây là trạng thái **enhance**: giữ nguyên `JOB`, `WORKER` từ base (JOB bỏ field `assigned_worker_id` trực tiếp, thay bằng quan hệ tới `JOB_CLAIM`), thêm mới:
- `JOB_CLAIM`: đại diện cho lock/lease theo từng job, có `fencing_token`, `lease_ttl_seconds`, `expires_at` để tự động hết hạn khi worker chết mà không release - đáp ứng yêu cầu 1; có `status` (HELD/RELEASED/EXPIRED) để phân biệt worker thắng/thua khi race - đáp ứng yêu cầu 2; có `renew_count`/`last_renewed_at` để theo dõi renew định kỳ - đáp ứng yêu cầu 3.
- `JOB` thêm field `result_applied` (idempotency marker) để kiểm tra trước khi ghi kết quả, tránh áp dụng 2 lần khi có worker khác nhận lại job - đáp ứng yêu cầu 3.
- `WORKER` thêm field `shutdown_signal_received_at` để thể hiện graceful shutdown chủ động release job - đáp ứng yêu cầu 4.
- `DEAD_JOB_SCAN_LOG`: bản ghi mỗi lần cơ chế quét định kỳ phát hiện job "đang xử lý" quá lâu so với TTL kỳ vọng - đáp ứng yêu cầu 5.

```mermaid
erDiagram
    WORKER ||--o{ JOB_CLAIM : "giữ/đã từng giữ"
    JOB ||--o{ JOB_CLAIM : "được claim qua các lần"
    JOB_CLAIM ||--o{ DEAD_JOB_SCAN_LOG : "bị phát hiện nếu quá hạn"

    WORKER {
        string worker_id PK
        string status
        datetime started_at
        datetime shutdown_signal_received_at
    }
    JOB {
        string job_id PK
        string job_type
        string payload_ref
        string status
        boolean result_applied
        datetime created_at
        datetime updated_at
    }
    JOB_CLAIM {
        string claim_id PK
        string job_id FK
        string worker_id FK
        string fencing_token
        datetime acquired_at
        int lease_ttl_seconds
        datetime expires_at
        int renew_count
        datetime last_renewed_at
        string status
    }
    DEAD_JOB_SCAN_LOG {
        string scan_id PK
        string job_id FK
        string claim_id FK
        datetime detected_at
        string action_taken
    }
```

---

# Sequence Diagram - Enhance: claim-and-process-job

Đây là flow **enhance** của `claim-and-process-job` (so với base ở file `2-base-claim-and-process-job-sqd.md`): 2 worker cùng thấy job "AVAILABLE" và cùng cố acquire lock/lease gần như đồng thời (race condition khi liệt kê job không atomic với acquire), nhưng thao tác claim thực tế là 1 lệnh atomic (compare-and-swap trên trạng thái job hoặc acquire trên coordinator) nên chỉ đúng 1 worker thắng và có TTL ngay từ lúc claim. Đáp ứng yêu cầu 1 (claim bằng lock/lease có TTL) và yêu cầu 2 (race condition chỉ 1 worker thắng, worker thua nhận biết ngay).

```mermaid
sequenceDiagram
    participant W1 as Worker 1
    participant W2 as Worker 2
    participant Q as Job Queue/DB (atomic claim)

    W1->>Q: Poll job status=AVAILABLE
    Q-->>W1: Trả về job J1
    W2->>Q: Poll job status=AVAILABLE (gần như cùng lúc)
    Q-->>W2: Trả về job J1
    par Cả 2 worker cùng cố claim J1
        W1->>Q: Atomic claim J1 (CAS status=AVAILABLE->PROCESSING, set lease_ttl=60s)
        W2->>Q: Atomic claim J1 (CAS status=AVAILABLE->PROCESSING, set lease_ttl=60s)
    end
    Q-->>W1: Thành công, cấp fencing_token=101, expires_at=+60s
    Q-->>W2: Thất bại, job đã ở PROCESSING (thua race)
    W2->>W2: Nhận biết ngay, không xử lý song song, quay lại poll job khác
    W1->>W1: Xử lý job J1
    W1->>Q: Update job J1 status=DONE, release claim
    Q-->>W1: OK
```

---

# Sequence Diagram - Enhance: lease-renew-and-idempotent-apply

Đây là flow **enhance mới**: job cần xử lý lâu hơn TTL ước tính (file lớn bất thường), worker phải renew lease định kỳ; lần renew giữa chừng thất bại do mất kết nối coordinator, lease hết hạn, worker khác nhận lại job trong khi worker cũ vẫn đang xử lý thật. Khi cả 2 cùng cố ghi kết quả, cơ chế kiểm tra idempotent (dựa trên `result_applied`/fencing_token) đảm bảo kết quả chỉ được áp dụng đúng 1 lần. Đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant W1 as Worker 1 (đang xử lý job lâu)
    participant Q as Job Queue/DB
    participant W2 as Worker 2

    W1->>Q: Claim job J1, lease_ttl=60s, fencing_token=201
    Q-->>W1: OK, expires_at=+60s
    loop Xử lý job kéo dài, renew định kỳ
        W1->>Q: Renew lease (gia hạn expires_at)
        Q-->>W1: OK
    end
    Note over W1,Q: Một lần renew bị mất kết nối tới coordinator, không tới nơi
    Q->>Q: Lease hết hạn vì không renew kịp
    Q->>Q: Trả job J1 về status=AVAILABLE
    W2->>Q: Poll job status=AVAILABLE
    Q-->>W2: Trả về job J1
    W2->>Q: Claim job J1, fencing_token=202
    Q-->>W2: OK, expires_at mới
    Note over W1: Worker 1 vẫn đang xử lý thật, không biết đã mất lease
    W1->>Q: Ghi kết quả job J1 kèm fencing_token=201
    Q->>Q: Kiểm tra fencing_token=201 đã cũ (không khớp token hiện hành 202)
    Q-->>W1: Từ chối ghi, báo lease đã mất, huỷ apply
    W2->>W2: Xử lý job J1 xong
    W2->>Q: Ghi kết quả job J1 kèm fencing_token=202, kiểm tra result_applied=false
    Q->>Q: result_applied=false, chấp nhận ghi, set result_applied=true
    Q-->>W2: OK, job J1 status=DONE
```

---

# Sequence Diagram - Enhance: graceful-shutdown-release

Đây là flow **enhance mới**: khi worker nhận tín hiệu terminate có báo trước từ autoscale (graceful shutdown signal, ví dụ SIGTERM), worker chủ động release ngay lock/job đang giữ thay vì để coordinator chờ tới khi TTL hết hạn, giúp job khả dụng lại nhanh hơn nhiều so với chờ timeout tự nhiên. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    participant AS as Autoscaler
    participant W as Worker (đang giữ job J1)
    participant Q as Job Queue/DB
    participant W2 as Worker khác

    AS->>W: Gửi graceful shutdown signal (sắp terminate)
    W->>W: Nhận signal, dừng nhận job mới
    W->>Q: Release claim job J1 ngay lập tức (status=AVAILABLE, xoá lease)
    Q-->>W: OK
    W->>AS: Xác nhận sẵn sàng bị terminate
    AS->>W: Terminate worker
    W2->>Q: Poll job status=AVAILABLE
    Q-->>W2: Trả về job J1 (khả dụng ngay, không phải chờ TTL 60s)
    W2->>Q: Claim job J1
    Q-->>W2: OK
```

---

# Sequence Diagram - Enhance: dead-job-scan

Đây là flow **enhance mới**: một tiến trình quét định kỳ (dead job scanner) chủ động rà soát các job đang ở trạng thái "PROCESSING" quá lâu so với TTL kỳ vọng, thay vì chỉ dựa hoàn toàn vào cơ chế TTL tự nhiên - dùng để phát hiện sớm rò rỉ lock hoặc lỗi hệ thống (ví dụ TTL không tự expire được do bug, clock skew). Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    participant Scanner as Dead Job Scanner (chạy định kỳ)
    participant Q as Job Queue/DB
    participant Alert as Alerting/Ops

    loop Mỗi chu kỳ quét (ví dụ mỗi 30s)
        Scanner->>Q: Tìm job status=PROCESSING có updated_at quá lâu so với lease_ttl_seconds kỳ vọng
        Q-->>Scanner: Danh sách job nghi ngờ (ví dụ J5 đã PROCESSING 10 phút dù TTL=60s)
        alt Lease của job đã thực sự hết hạn nhưng job chưa được trả về AVAILABLE
            Scanner->>Q: Ghi log DEAD_JOB_SCAN_LOG, cưỡng chế trả job J5 về status=AVAILABLE
            Q-->>Scanner: OK
            Scanner->>Alert: Cảnh báo rò rỉ lock/lỗi hệ thống cho người vận hành
        else Lease vẫn còn hạn, chỉ là job xử lý lâu hợp lệ
            Scanner->>Q: Ghi log DEAD_JOB_SCAN_LOG với action_taken=no-op, không can thiệp
        end
    end
```
