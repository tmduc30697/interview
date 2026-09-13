# Sequence Diagram — Base: Checkout Create Order

Đây là **base**, flow "tạo đơn hàng lúc checkout" — tiền đề bắt buộc cho việc tách bảng: chính flow này đang ghi toàn bộ danh sách item vào cột `items_json` của bảng `orders`, là nguồn dữ liệu duy nhất cần được dual-write sang bảng `order_items` mới ở enhance.

```mermaid
sequenceDiagram
    actor Customer
    participant Checkout as Checkout Service
    participant OrderSvc as Order Service
    participant DB as Orders Table

    Customer->>Checkout: Submit checkout for cart
    Checkout->>Checkout: Validate cart items, compute total
    Checkout->>OrderSvc: Create order (cart items, total)
    OrderSvc->>DB: INSERT INTO orders (items_json, total_amount, status)
    DB-->>OrderSvc: order_id
    OrderSvc-->>Checkout: Order created
    Checkout-->>Customer: Checkout success, order confirmation
```
