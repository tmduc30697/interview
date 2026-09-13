# Sequence Diagram - Enhance: checkout-order

Đây là flow **enhance** của `checkout-order` (so với base ở file `2-base-checkout-order-sqd.md`): `trace_id` được giữ nguyên xuyên suốt cả 4 bước, kể cả đoạn `payment-service` phải chờ callback bất đồng bộ từ cổng thanh toán thay vì trả lời đồng bộ ngay, giúp dựng lại được toàn bộ hành trình của 1 đơn hàng cụ thể khi khách hàng báo lỗi. Đáp ứng yêu cầu 1.

```mermaid
sequenceDiagram
    actor Customer
    participant Cart as cart-service
    participant Inv as inventory-service
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài
    participant Confirm as order-confirmation-service

    Customer->>Cart: Checkout giỏ hàng
    Cart->>Cart: Sinh trace_id=TR9, tạo span S1
    Cart->>Inv: Yêu cầu giữ tồn kho, header trace_id=TR9, parent_span_id=S1
    Inv->>Inv: Tạo span S2 (trace_id=TR9, parent_span_id=S1)
    Inv-->>Cart: Giữ tồn kho thành công, kết thúc span S2
    Cart->>Pay: Yêu cầu thanh toán, header trace_id=TR9, parent_span_id=S1
    Pay->>Pay: Tạo span S3 (trace_id=TR9, parent_span_id=S1)
    Pay->>Gateway: Gọi cổng thanh toán, kèm trace_id=TR9 trong metadata request
    Note over Pay,Gateway: payment-service không chờ đồng bộ, span S3 tạm ở trạng thái pending callback
    Gateway-->>Pay: Callback bất đồng bộ báo thanh toán thành công, kèm trace_id=TR9
    Pay->>Pay: Kết thúc span S3 khi nhận callback
    Pay-->>Cart: Thanh toán OK
    Cart->>Confirm: Yêu cầu xác nhận đơn hàng, header trace_id=TR9, parent_span_id=S1
    Confirm->>Confirm: Tạo span S4 (trace_id=TR9, parent_span_id=S1)
    Confirm-->>Customer: Đơn hàng đã xác nhận, kết thúc span S4 và trace TR9
```
