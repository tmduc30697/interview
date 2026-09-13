# Base sequence — Create shipment (chưa có circuit breaker, retry mù)

Đây là **base**, flow "tạo vận đơn" ở trạng thái trước enhance. Hệ thống chỉ gọi thẳng carrier được gán cho đơn hàng, không có breaker, không có carrier dự phòng, và khi timeout thì retry mù ngay lập tức mà không xác minh trạng thái thật ở phía carrier. Flow này là nền để so sánh với flow cùng tên ở enhance (per-carrier breaker, xác minh trước khi retry, fallback, jitter).

```mermaid
sequenceDiagram
    actor OrderSvc as Order Service
    participant CarrierA as Carrier A API (create)
    participant DB as Shipment DB

    OrderSvc->>DB: Tạo bản ghi SHIPMENT status=pending, carrier=A
    OrderSvc->>CarrierA: Gọi API tạo vận đơn cho đơn hàng
    alt Thành công
        CarrierA-->>OrderSvc: Trả về tracking_code
        OrderSvc->>DB: Cập nhật SHIPMENT status=created, ghi tracking_code
    else Timeout (không rõ carrier đã tạo vận đơn hay chưa)
        CarrierA-->>OrderSvc: Không có phản hồi trong thời gian chờ
        Note over OrderSvc: Không kiểm tra lại trạng thái thật, chỉ dựa vào timeout
        OrderSvc->>CarrierA: Retry ngay lập tức, không backoff, không jitter
        Note over OrderSvc,CarrierA: Rủi ro, nếu lần gọi trước đã thực sự tạo vận đơn thì lần này tạo trùng
        CarrierA-->>OrderSvc: Kết quả lần retry (có thể lại timeout hoặc tạo trùng vận đơn)
    else Lỗi rõ ràng từ carrier (4xx/5xx)
        CarrierA-->>OrderSvc: Trả lỗi
        OrderSvc->>DB: Cập nhật SHIPMENT status=failed
    end
```
