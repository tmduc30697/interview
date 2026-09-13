# Sequence Diagram - Base: checkout-order

Đây là flow **base**: đơn hàng đi qua `cart-service` -> `inventory-service` -> `payment-service` (gọi cổng thanh toán bên ngoài) -> `order-confirmation-service`, mỗi service chỉ ghi log cục bộ của riêng nó. Flow này là nền để so sánh với enhance, nơi mỗi bước sẽ được gắn `trace_id`/`span` xuyên suốt kể cả đoạn chờ callback bất đồng bộ.

```mermaid
sequenceDiagram
    actor Customer
    participant Cart as cart-service
    participant Inv as inventory-service
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài
    participant Confirm as order-confirmation-service

    Customer->>Cart: Checkout giỏ hàng
    Cart->>Cart: Ghi log cục bộ "tạo order"
    Cart->>Inv: Yêu cầu giữ tồn kho cho order
    Inv->>Inv: Ghi log cục bộ "giữ tồn kho"
    Inv-->>Cart: Giữ tồn kho thành công
    Cart->>Pay: Yêu cầu thanh toán
    Pay->>Gateway: Gọi cổng thanh toán bên ngoài
    Gateway-->>Pay: Thanh toán thành công
    Pay->>Pay: Ghi log cục bộ "thanh toán thành công"
    Pay-->>Cart: Thanh toán OK
    Cart->>Confirm: Yêu cầu xác nhận đơn hàng
    Confirm->>Confirm: Ghi log cục bộ "xác nhận đơn hàng"
    Confirm-->>Customer: Đơn hàng đã xác nhận
```
