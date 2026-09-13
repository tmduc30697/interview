# Base sequence — Track shipment status

Đây là **base**, flow tra cứu trạng thái vận đơn qua API của chính carrier đã tạo đơn đó. Flow này vốn đã tồn tại độc lập (dùng để cập nhật trạng thái giao hàng cho khách/nội bộ) và ở base chưa được dùng cho mục đích nào khác. Flow này quan trọng vì ở enhance, chính API tra cứu này sẽ được tái sử dụng để xác minh trước khi quyết định retry (yêu cầu 3), thay vì retry mù như flow tạo vận đơn ở base.

```mermaid
sequenceDiagram
    actor OrderSvc as Order Service
    participant CarrierA as Carrier A API (track)
    participant DB as Shipment DB

    OrderSvc->>DB: Lấy tracking_code / order reference của SHIPMENT
    OrderSvc->>CarrierA: Gọi API tra cứu trạng thái theo tracking_code hoặc order reference
    CarrierA-->>OrderSvc: Trả về trạng thái hiện tại, ví dụ chưa tạo, đã tạo, đang giao, đã giao
    OrderSvc->>DB: Cập nhật SHIPMENT status theo kết quả tra cứu
```
