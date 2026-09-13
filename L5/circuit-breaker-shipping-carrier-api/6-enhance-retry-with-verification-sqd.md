# Enhance sequence — Retry an toàn sau timeout (xác minh trước khi retry, backoff có jitter)

Đây là **enhance**, flow hoàn toàn mới, mô tả chi tiết điều gì xảy ra khi một lần gọi API tạo vận đơn bị timeout (khác với trường hợp breaker đã mở ở flow create-shipment). Đáp ứng yêu cầu 2 (tránh tạo trùng vận đơn khi thực ra carrier A đã tạo thành công), yêu cầu 3 (chỉ retry sau khi xác minh qua API tra cứu của chính carrier đó rằng vận đơn thực sự chưa được tạo), và yêu cầu 4 (backoff giữa các lần retry có jitter để tránh dồn request vào carrier lúc cao điểm).

```mermaid
sequenceDiagram
    actor OrderSvc as Order Service
    participant CarrierA as Carrier A API (create)
    participant CarrierALookup as Carrier A API (track)
    participant BreakerA as Breaker (Carrier A)
    participant CarrierB as Carrier B API

    OrderSvc->>CarrierA: Gọi API tạo vận đơn
    CarrierA-->>OrderSvc: Timeout, không rõ đã tạo hay chưa
    Note over OrderSvc: Không retry mù, phải xác minh trước qua API tra cứu của chính Carrier A
    OrderSvc->>CarrierALookup: Tra cứu trạng thái vận đơn theo order reference

    alt Vận đơn ĐÃ thực sự được tạo (timeout chỉ là phản hồi chậm)
        CarrierALookup-->>OrderSvc: Đã tồn tại, kèm tracking_code
        OrderSvc->>OrderSvc: Ghi nhận thành công với tracking_code đã có, không tạo lại
    else Vận đơn CHƯA được tạo, xác nhận an toàn để retry
        CarrierALookup-->>OrderSvc: Chưa tồn tại
        OrderSvc->>OrderSvc: Tính thời gian chờ = backoff_base_ms nhân theo số lần thử, cộng thêm jitter ngẫu nhiên
        Note over OrderSvc: Jitter giúp tránh nhiều request retry cùng dồn vào Carrier A đúng lúc carrier đang cao điểm hoặc mới phục hồi
        OrderSvc->>CarrierA: Retry gọi API tạo vận đơn sau khi chờ đủ thời gian backoff có jitter
        alt Retry thành công
            CarrierA-->>OrderSvc: Trả về tracking_code
        else Vẫn timeout hoặc lỗi liên tiếp vượt ngưỡng
            OrderSvc->>BreakerA: Ghi nhận lỗi liên tiếp, breaker Carrier A chuyển sang open
            OrderSvc->>CarrierALookup: Xác minh lần cuối, chắc chắn Carrier A chưa tạo vận đơn
            CarrierALookup-->>OrderSvc: Xác nhận chưa tạo
            OrderSvc->>CarrierB: Chuyển sang tạo vận đơn ở carrier dự phòng, an toàn vì đã xác minh không trùng đơn
        end
    end
```
