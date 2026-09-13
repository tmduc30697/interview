# Checkout gọi payment gateway bên thứ ba

**Hệ thống:** Trang e-commerce gọi API cổng thanh toán bên ngoài (kiểu Stripe) để xử lý thanh toán lúc checkout.

**Vai trò của flow:** Circuit breaker + retry with backoff bảo vệ hệ thống khi payment gateway chậm/lỗi, tránh làm nghẽn toàn bộ luồng checkout và tránh gọi lại dồn dập làm gateway càng quá tải.

**Yêu cầu cụ thể:**
- Định nghĩa rõ ngưỡng mở circuit breaker (ví dụ tỉ lệ lỗi >50% trong 10 giây gần nhất, hoặc latency p95 vượt X ms) và hành vi khi breaker mở (fail-fast, không gọi gateway nữa trong một khoảng thời gian).
- Retry chỉ áp dụng cho lỗi có thể retry an toàn (timeout, 5xx) — tuyệt đối không retry tự động cho response không rõ kết quả liên quan tới việc charge tiền (tránh double-charge), phải có bước kiểm tra idempotency key trước khi retry.
- Backoff phải là exponential kèm jitter (ngẫu nhiên hóa) để tránh nhiều request cùng retry đồng loạt tạo ra "retry storm" làm gateway sập thêm.
- Circuit breaker phải có trạng thái half-open để thử lại một lượng nhỏ request sau khi hết thời gian mở, và chỉ đóng lại hoàn toàn khi các request thử nghiệm đó thành công.
- Đo lường: tỉ lệ request bị fail-fast do breaker mở, số lần breaker chuyển trạng thái trong ngày, và alert khi breaker mở quá lâu (báo hiệu sự cố kéo dài ở phía gateway).

---

# Base ERD — Checkout gọi payment gateway trước khi có circuit breaker

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có circuit breaker + retry an toàn. Đề bài nói checkout gọi trực tiếp API cổng thanh toán bên ngoài — nên base cần đủ: đơn hàng và các lần gọi thanh toán, ghi nhận đơn giản không phân loại lỗi. Chưa có entity nào phục vụ circuit breaker state/idempotency key/retry policy/metric — những thứ đó là phần enhance.

```mermaid
erDiagram
    ORDER ||--o{ PAYMENT_ATTEMPT : "attempts payment via"

    ORDER {
        string id PK
        string status "pending | paid | failed"
    }
    PAYMENT_ATTEMPT {
        string id PK
        string order_id FK
        string gateway_request_id
        string status "success | failed | timeout"
        datetime attempted_at
    }
```

---

# Base sequence — Checkout payment (gọi trực tiếp, retry mù quáng)

Đây là **base**, flow "Thanh toán lúc checkout" ở trạng thái hiện tại — gọi thẳng gateway, retry ngay lập tức không phân biệt loại lỗi, không kiểm tra idempotency. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng các lỗ hổng ở đây.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant Gateway as Payment Gateway (bên thứ ba)
    participant DB as PAYMENT_ATTEMPT store

    Customer->>App: Xác nhận thanh toán
    App->>Gateway: Gọi API charge
    Gateway-->>App: Timeout (không rõ đã charge hay chưa)
    App->>DB: Ghi PAYMENT_ATTEMPT(status=timeout)
    App->>Gateway: Retry ngay lập tức (không kiểm tra idempotency, không phân biệt lỗi có an toàn để retry hay không)
    Gateway-->>App: Charge thành công (lần 2)
    Note over Gateway,DB: Nếu lần gọi đầu thực ra đã charge thành công phía gateway trước khi timeout trả về, khách hàng đã bị charge 2 lần
    App->>DB: Ghi PAYMENT_ATTEMPT(status=success)
    App-->>Customer: "Thanh toán thành công"
    Note over App,Gateway: Khi gateway đang chậm/lỗi hàng loạt, mọi request checkout đều tự retry dồn dập, làm gateway càng quá tải hơn
```

---

# Enhance ERD — sau khi có circuit breaker + retry an toàn

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `CIRCUIT_BREAKER_STATE` (mới) — trạng thái closed/open/half_open, ngưỡng mở, thời điểm mở.
- `IDEMPOTENCY_KEY` (mới) — gắn với mỗi lần charge, dùng để kiểm tra trước khi quyết định retry cho lỗi không rõ kết quả.
- `RETRY_POLICY` (mới) — phân loại rõ lỗi nào được retry, backoff exponential kèm jitter.
- `BREAKER_METRIC` + `BREAKER_ALERT` (mới) — đo tỉ lệ fail-fast, số lần chuyển trạng thái, cảnh báo khi breaker mở quá lâu.

```mermaid
erDiagram
    ORDER ||--o{ PAYMENT_ATTEMPT : "attempts payment via"
    PAYMENT_ATTEMPT ||--|| IDEMPOTENCY_KEY : "tagged with"
    PAYMENT_ATTEMPT ||--|| RETRY_POLICY : "governed by"
    CIRCUIT_BREAKER_STATE ||--o{ BREAKER_METRIC : measured
    CIRCUIT_BREAKER_STATE ||--o{ BREAKER_ALERT : "may trigger"

    ORDER {
        string id PK
        string status "pending | paid | failed"
    }
    PAYMENT_ATTEMPT {
        string id PK
        string order_id FK
        string gateway_request_id
        string status "success | failed | timeout | unknown"
        string error_type "timeout | 5xx | unknown_ambiguous"
        int attempt_number
        datetime attempted_at
    }
    IDEMPOTENCY_KEY {
        string id PK
        string order_id FK
        string key
        string gateway_response_status "success | failed | unknown"
        datetime created_at
    }
    RETRY_POLICY {
        string id PK
        string error_type "timeout | 5xx | unknown_ambiguous"
        boolean retryable
        int backoff_base_ms
        boolean jitter_enabled
    }
    CIRCUIT_BREAKER_STATE {
        string id PK
        string service_name "payment_gateway"
        string state "closed | open | half_open"
        decimal error_rate_threshold
        int error_rate_window_seconds
        datetime opened_at
        int half_open_test_count
    }
    BREAKER_METRIC {
        string id PK
        string service_name FK
        datetime window_start
        datetime window_end
        int fail_fast_count
        int state_transition_count
    }
    BREAKER_ALERT {
        string id PK
        string service_name FK
        datetime opened_at
        int duration_ms
        datetime alert_triggered_at
    }
```

---

# Enhance sequence — Checkout payment (qua circuit breaker, retry an toàn)

Đây là **enhance**, cùng flow "Checkout payment" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1, 2 và 3 của đề bài: gọi qua circuit breaker, chỉ retry lỗi an toàn với exponential backoff kèm jitter, và kiểm tra idempotency trước khi retry lỗi không rõ kết quả.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant Breaker as CIRCUIT_BREAKER_STATE
    participant Gateway as Payment Gateway
    participant Idem as IDEMPOTENCY_KEY store
    participant Policy as RETRY_POLICY store

    Customer->>App: Xác nhận thanh toán
    App->>Breaker: Kiểm tra state
    alt Breaker = open
        Breaker-->>App: Fail-fast ngay, không gọi Gateway
        App-->>Customer: "Cổng thanh toán đang gặp sự cố, vui lòng thử lại sau"
    else Breaker = closed hoặc half_open
        App->>Idem: Tạo IDEMPOTENCY_KEY cho lần charge này
        App->>Gateway: Gọi API charge kèm idempotency key
        Gateway-->>App: Kết quả (success / 5xx / timeout)
        alt Kết quả rõ ràng là lỗi có thể retry an toàn (timeout, 5xx)
            App->>Policy: Kiểm tra retryable=true, lấy backoff_base_ms + jitter
            App->>App: Chờ theo exponential backoff + jitter ngẫu nhiên
            App->>Gateway: Retry cùng idempotency key
            Gateway-->>App: Gateway nhận diện idempotency key trùng, trả kết quả nhất quán (không charge lần 2)
        else Kết quả không rõ ràng (unknown_ambiguous)
            App->>Idem: Tra cứu lại trạng thái gateway_response_status theo idempotency key trước khi quyết định
            Idem-->>App: Trạng thái xác nhận thực tế (success/failed)
            Note over App: Tuyệt đối không tự động retry ngay khi chưa xác minh — tránh double-charge
        end
        App-->>Customer: Kết quả thanh toán chính xác
    end
```

---

# Enhance sequence — Circuit breaker state machine

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm breaker). Đáp ứng yêu cầu 1 và 4 của đề bài: ngưỡng mở rõ ràng, hành vi fail-fast khi mở, và trạng thái half-open thử lại một lượng nhỏ request trước khi đóng hẳn.

```mermaid
sequenceDiagram
    participant Breaker as CIRCUIT_BREAKER_STATE
    participant Gateway as Payment Gateway
    actor Request as Request thanh toán bất kỳ

    Note over Breaker: state=closed (bình thường)
    loop Theo dõi liên tục
        Request->>Gateway: Gọi thật (breaker closed cho qua)
        Gateway-->>Breaker: Cập nhật tỉ lệ lỗi/latency trong window gần nhất
    end

    alt Tỉ lệ lỗi >50% trong 10 giây gần nhất, hoặc latency p95 vượt ngưỡng
        Breaker->>Breaker: Chuyển state=open, ghi opened_at=now
        Note over Breaker: Toàn bộ request tiếp theo bị fail-fast ngay, không gọi Gateway nữa
    end

    Note over Breaker: Sau khoảng thời gian mở cố định
    Breaker->>Breaker: Chuyển state=half_open
    loop Với 1 lượng nhỏ request thử nghiệm
        Request->>Gateway: Cho phép gọi thật (số lượng giới hạn)
        Gateway-->>Breaker: Kết quả request thử
    end
    alt Toàn bộ request thử nghiệm thành công
        Breaker->>Breaker: Chuyển state=closed, reset bộ đếm lỗi
    else Có request thử nghiệm thất bại
        Breaker->>Breaker: Quay lại state=open, reset lại thời gian mở
    end
```

---

# Enhance sequence — Breaker metrics & alerting

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: đo tỉ lệ fail-fast, số lần breaker chuyển trạng thái trong ngày, và alert khi breaker mở quá lâu.

```mermaid
sequenceDiagram
    participant Breaker as CIRCUIT_BREAKER_STATE
    participant App as Checkout Service
    participant Metric as BREAKER_METRIC store
    participant AlertJob as Alert Monitor
    participant Alert as BREAKER_ALERT store
    actor OnCall as On-call Engineer

    App->>Metric: Mỗi lần fail-fast do breaker open, tăng fail_fast_count trong window
    Breaker->>Metric: Mỗi lần chuyển state (closed→open, open→half_open, half_open→closed/open), tăng state_transition_count

    loop Định kỳ theo dõi
        AlertJob->>Breaker: Kiểm tra state hiện tại và opened_at
        alt state=open và đã mở quá lâu (vượt ngưỡng thời gian cho phép)
            AlertJob->>Alert: Ghi BREAKER_ALERT(duration_ms, alert_triggered_at)
            Alert-->>OnCall: Cảnh báo sự cố kéo dài ở phía gateway, cần điều tra
        else Trong ngưỡng bình thường
            AlertJob->>AlertJob: Không làm gì thêm
        end
    end

    Metric-->>OnCall: Báo cáo cuối ngày: tỉ lệ request bị fail-fast, số lần breaker đổi trạng thái trong ngày
```
