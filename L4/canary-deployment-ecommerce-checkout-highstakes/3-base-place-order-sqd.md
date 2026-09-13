# Base sequence — Place order

Đây là **base**, flow "Đặt hàng/checkout" — nền tảng cho enhance: đơn hàng không gắn với version nào đã xử lý nó, nên khi có nhiều version chạy song song (canary), hệ thống chưa có cách xác định và xử lý nhất quán 1 đơn đang dở dang ở đúng version nào.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant Inv as Inventory
    participant Gateway as Payment Gateway
    participant DB as ORDER / PAYMENT_INTENT store

    Customer->>App: Xác nhận đặt hàng
    App->>Inv: Giữ tồn kho cho đơn hàng
    Inv-->>App: Giữ thành công
    App->>DB: Tạo ORDER(status=pending, inventory_reserved=true)
    App->>Gateway: Tạo payment intent, thực hiện charge
    Gateway-->>App: Kết quả thanh toán
    App->>DB: Cập nhật ORDER.status theo kết quả (completed/failed)
    App-->>Customer: Kết quả đặt hàng
```
