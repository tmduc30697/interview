# Checkout service của sàn e-commerce trong lúc deploy

**Hệ thống:** Một service xử lý checkout (tạo order, trừ tồn kho, gọi cổng thanh toán) cho sàn e-commerce, được deploy nhiều lần mỗi ngày.

**Vai trò của flow:** Khi rolling deploy loại bỏ instance cũ, instance đó phải hoàn tất các giao dịch checkout đang xử lý trước khi tắt, không được cắt ngang giữa lúc đang trừ tồn kho hoặc gọi cổng thanh toán.

**Yêu cầu cụ thể:**
- Khi nhận SIGTERM, instance phải ngay lập tức báo readiness = false (để load balancer ngừng gửi request mới) nhưng vẫn tiếp tục xử lý các request đang chạy cho tới khi xong hoặc hết grace period.
- Định nghĩa rõ grace period tối đa (ví dụ 30 giây) — nếu một giao dịch checkout chưa xong khi hết grace period, phải có chiến lược rõ ràng (log lại state, không để order ở trạng thái "nửa vời" — ví dụ đã trừ tồn kho nhưng chưa tạo order) thay vì kill cứng.
- Xử lý trường hợp instance đang giữ một distributed lock (ví dụ lock tồn kho sản phẩm) khi bị shutdown — phải release lock hoặc để lock tự hết TTL, không được để lock treo vĩnh viễn làm nghẽn các checkout khác.
- Đảm bảo trong lúc drain, instance không nhận thêm request mới (health check đã báo not-ready) nhưng vẫn phải trả lời được health check probe để không bị hạ tầng kill ngay lập tức trước khi drain xong.
- Viết test mô phỏng gửi 100 request checkout đồng thời rồi trigger shutdown giữa chừng, verify không có request nào bị mất kết nối nửa đường (mỗi request phải nhận được response, kể cả response lỗi rõ ràng, không bao giờ là timeout im lặng).

---

# Base ERD — Checkout service trước khi có graceful shutdown

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** instance biết xử lý shutdown an toàn. Đề bài nói checkout tạo order, trừ tồn kho, gọi cổng thanh toán, và có nhắc tới việc instance có thể đang giữ distributed lock tồn kho — nên base cần đủ: sản phẩm/tồn kho, đơn hàng, distributed lock dùng khi trừ tồn kho. Chưa có khái niệm readiness/draining, chưa có grace period, chưa có chiến lược xử lý đơn nửa vời, chưa có kịch bản test toàn trình — tất cả là phần enhance.

```mermaid
erDiagram
    PRODUCT ||--o{ ORDER : "ordered as"
    PRODUCT ||--o| DISTRIBUTED_LOCK : "locked by"

    PRODUCT {
        string id PK
        int stock
    }
    ORDER {
        string id PK
        string product_id FK
        string status "pending | paid | failed"
        datetime created_at
    }
    DISTRIBUTED_LOCK {
        string lock_key PK
        string holder_instance_id
        datetime acquired_at
        datetime ttl_expires_at
    }
```

---

# Base sequence — Checkout (kill cứng khi deploy, lock treo, đơn nửa vời)

Đây là **base**, flow "Checkout: trừ tồn kho, tạo order, gọi cổng thanh toán" ở trạng thái hiện tại — deploy đơn giản kill cứng instance ngay khi có bản mới, không có bước báo readiness=false, không có grace period, không có xử lý khi đang giữ lock. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng khoảng hở này.

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer
    participant App as Checkout Instance A
    participant DB as PRODUCT/ORDER store
    participant Gateway as Payment Gateway

    Customer->>LB: Gửi request checkout
    LB->>App: Route request (App vẫn đang nhận traffic bình thường)
    App->>DB: Acquire DISTRIBUTED_LOCK(product_id)
    App->>DB: UPDATE PRODUCT SET stock = stock - 1
    App->>DB: INSERT ORDER (status=pending)

    Note over App: Đúng lúc này, hệ thống deploy trigger kill cứng process ngay lập tức, không có bước drain nào
    App--xApp: Process bị chấm dứt giữa lúc đang gọi Payment Gateway

    Note over DB,Gateway: Tồn kho đã bị trừ, ORDER đã tạo ở status=pending, nhưng không rõ Gateway đã charge hay chưa, và DISTRIBUTED_LOCK không được release, cứ treo cho tới khi tự hết TTL
    Customer->>LB: Không nhận được response nào, client timeout im lặng, không biết đơn có thành công hay không
```

---

# Enhance ERD — Thêm readiness/draining, chiến lược đơn nửa vời, kịch bản test toàn trình

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi:
- Entity mới `INSTANCE` — đại diện 1 instance của checkout service, có `readiness`, `status` (ready/draining/stopped), `grace_period_seconds`, `shutdown_deadline_at` — đáp ứng yêu cầu 1, 2 và 4 (báo readiness=false ngay, nhưng vẫn trả lời health check probe trong lúc drain).
- `ORDER` có thêm trạng thái `inventory_reserved` và `failed_incomplete` cùng cột `incomplete_reason` — đáp ứng yêu cầu 2 (khi hết grace period mà giao dịch chưa xong, ghi lại state rõ ràng thay vì để đơn "nửa vời" không rõ nguyên nhân).
- `DISTRIBUTED_LOCK` có thêm cột `released_at` và `released_by` — đáp ứng yêu cầu 3 (đảm bảo lock được release hoặc tự hết TTL, không treo vĩnh viễn).
- Entity mới `CHECKOUT_LOAD_TEST_RUN` — ghi nhận kết quả bài test toàn trình gửi 100 request đồng thời rồi trigger shutdown giữa chừng — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    PRODUCT ||--o{ ORDER : "ordered as"
    PRODUCT ||--o| DISTRIBUTED_LOCK : "locked by"
    INSTANCE ||--o{ ORDER : handles
    INSTANCE ||--o{ DISTRIBUTED_LOCK : holds
    CHECKOUT_LOAD_TEST_RUN ||--o{ ORDER : observes

    PRODUCT {
        string id PK
        int stock
    }
    ORDER {
        string id PK
        string product_id FK
        string instance_id FK
        string status "pending | inventory_reserved | paid | completed | failed_incomplete"
        string incomplete_reason "vd grace_period_expired_during_payment_call"
        datetime created_at
    }
    DISTRIBUTED_LOCK {
        string lock_key PK
        string holder_instance_id FK
        datetime acquired_at
        datetime ttl_expires_at
        datetime released_at
        string released_by "explicit | ttl_expired"
    }
    INSTANCE {
        string id PK
        boolean readiness
        string status "ready | draining | stopped"
        int grace_period_seconds
        datetime shutdown_deadline_at
    }
    CHECKOUT_LOAD_TEST_RUN {
        string id PK
        int total_requests
        int responses_received
        int silent_timeouts
        string status "running | passed | failed"
    }
```

---

# Enhance sequence — Checkout (readiness=false ngay lập tức, hoàn tất giao dịch đang chạy, release lock)

Đây là **enhance** của flow `checkout` đã có ở base, đáp ứng **yêu cầu 1 và yêu cầu 3** của đề bài. So với base: khi nhận SIGTERM, instance báo `readiness=false` ngay lập tức để load balancer ngừng gửi request mới, nhưng vẫn tiếp tục xử lý request đang chạy cho tới khi xong; sau khi giao dịch hoàn tất, `DISTRIBUTED_LOCK` được release tường minh thay vì để treo.

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer
    participant App as Checkout Instance A
    participant DB as PRODUCT/ORDER/LOCK store
    participant Gateway as Payment Gateway

    Customer->>LB: Gửi request checkout (request X)
    LB->>App: Route request X (App vẫn ready)
    App->>DB: Acquire DISTRIBUTED_LOCK(product_id)
    App->>DB: UPDATE PRODUCT SET stock=stock-1, INSERT ORDER(status=inventory_reserved)

    Note over App: Nhận SIGTERM ngay lúc này, giữa lúc request X đang chạy
    App->>LB: Báo readiness=false ngay lập tức (không đợi request X xong)
    LB->>LB: Ngừng route request mới tới App, các request đang chạy dở (như X) không bị huỷ

    App->>Gateway: Tiếp tục gọi charge cho request X (không bị cắt ngang)
    Gateway-->>App: Charge thành công
    App->>DB: UPDATE ORDER SET status=completed
    App->>DB: Release DISTRIBUTED_LOCK(product_id), released_by=explicit
    App-->>Customer: 200 OK, đơn hàng hoàn tất

    App->>App: Không còn request nào đang xử lý dở, instance tắt hẳn (status=stopped)
```

---

# Enhance sequence — Hết grace period giữa lúc giao dịch chưa xong, ghi state rõ ràng thay vì kill cứng

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 2 và phần còn lại của yêu cầu 3** của đề bài: định nghĩa grace period tối đa (30 giây) — nếu giao dịch checkout chưa xong khi hết grace period, hệ thống ghi lại state rõ ràng (`failed_incomplete` kèm lý do) thay vì kill cứng để lại đơn nửa vời, đồng thời đảm bảo `DISTRIBUTED_LOCK` được release tường minh trước khi tắt hẳn.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Instance A (đang drain, grace_period=30s)
    participant DB as PRODUCT/ORDER/LOCK store
    participant Gateway as Payment Gateway (đang phản hồi chậm)

    Note over App: Nhận SIGTERM lúc t=0, readiness=false, shutdown_deadline_at=t+30s
    Customer->>App: Request checkout Y đang xử lý dở từ trước SIGTERM
    App->>DB: Acquire DISTRIBUTED_LOCK(product_id), UPDATE stock, INSERT ORDER(status=inventory_reserved)
    App->>Gateway: Gọi charge cho request Y
    Note over Gateway: Gateway phản hồi rất chậm, không trả lời kịp

    Note over App: t=30s, hết grace period, request Y vẫn chưa có kết quả từ Gateway
    App->>DB: UPDATE ORDER SET status=failed_incomplete, incomplete_reason="grace_period_expired_during_payment_call"
    App->>DB: Release DISTRIBUTED_LOCK(product_id), released_by=explicit (không để treo cho tới khi tự hết TTL)
    App-->>Customer: 503 kèm order_id, giải thích giao dịch chưa xác nhận được, cần kiểm tra lại trạng thái đơn sau
    App->>App: Tắt hẳn (status=stopped)

    Note over DB: Vì stock đã bị trừ nhưng ORDER=failed_incomplete được ghi rõ, job đối soát nền có thể phát hiện và tự động hoàn lại tồn kho nếu Gateway xác nhận sau đó là chưa charge thành công
```

---

# Enhance sequence — Health check probe vẫn được trả lời trong lúc drain

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 4** của đề bài: trong lúc drain, instance không nhận thêm request mới (health check báo not-ready) nhưng vẫn phải trả lời được health check probe, để hạ tầng (Kubernetes/orchestrator) không kill instance ngay lập tức trước khi drain xong.

```mermaid
sequenceDiagram
    participant Infra as Orchestrator (vd Kubernetes)
    participant LB as Load Balancer
    participant App as Checkout Instance A (đang drain)

    Note over App: Nhận SIGTERM, chuyển status=draining, readiness=false
    loop Mỗi vài giây trong suốt grace period
        Infra->>App: GET /healthz/liveness
        App-->>Infra: 200 OK (process vẫn sống, đang xử lý nốt request dở, chưa nên bị kill)
        LB->>App: GET /healthz/readiness
        App-->>LB: 503 Not Ready (readiness=false, ngừng nhận request mới)
        LB->>LB: Không route request mới tới App, nhưng vẫn giữ App trong danh sách theo dõi
    end

    Note over App: App tiếp tục xử lý các request đang chạy dở song song với việc trả lời liveness probe đều đặn
    App->>App: Xử lý xong toàn bộ request đang chạy dở (hoặc hết grace period, xem flow grace-period-timeout-incomplete-order)
    App->>Infra: Chủ động thoát tiến trình (exit code 0)
    Infra->>Infra: Nhận diện process tự thoát sạch, không cần gửi SIGKILL cưỡng chế
```

---

# Enhance sequence — Test toàn trình 100 request đồng thời rồi trigger shutdown giữa chừng

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: gửi 100 request checkout đồng thời rồi trigger shutdown giữa chừng, verify không có request nào bị mất kết nối nửa đường — mỗi request phải nhận được response (kể cả response lỗi rõ ràng như 503/failed_incomplete), không bao giờ là timeout im lặng.

```mermaid
sequenceDiagram
    participant Test as Test Orchestrator
    participant LoadGen as Load Generator
    participant LB as Load Balancer
    participant App as Checkout Instance A
    participant Run as CHECKOUT_LOAD_TEST_RUN

    Test->>Run: INSERT CHECKOUT_LOAD_TEST_RUN(total_requests=100, status=running)
    Test->>LoadGen: Bắn 100 request checkout đồng thời qua LB tới App
    LoadGen->>LB: 100 request song song
    LB->>App: Route cả 100 request (App đang readiness=true)

    Note over Test: Giữa lúc 100 request đang xử lý dở (một số đã xong, một số đang gọi Gateway), Test trigger SIGTERM cho App
    Test->>App: Gửi SIGTERM
    App->>LB: Báo readiness=false ngay lập tức
    LB->>LB: Ngừng route request mới, 100 request đang chạy dở không bị huỷ

    loop Với từng request trong 100 request đang xử lý dở
        alt Xử lý xong trước khi hết grace period
            App-->>LoadGen: Response 200 (thành công)
        else Hết grace period mà chưa xong
            App-->>LoadGen: Response 503 kèm order failed_incomplete (lỗi rõ ràng, không phải timeout im lặng)
        end
    end

    LoadGen->>Test: Tổng hợp: responses_received=100, silent_timeouts=0
    Test->>Run: UPDATE responses_received=100, silent_timeouts=0

    alt silent_timeouts == 0 và mọi request đều có response
        Test->>Run: UPDATE status=passed
    else Có request không nhận được response nào (timeout im lặng)
        Test->>Run: UPDATE status=failed
    end
```
