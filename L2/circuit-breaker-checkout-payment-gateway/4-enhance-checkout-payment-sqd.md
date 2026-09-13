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
