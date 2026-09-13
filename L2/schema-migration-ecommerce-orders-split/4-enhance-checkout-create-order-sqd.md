# Sequence Diagram — Enhance: Checkout Create Order

Đây là **enhance**, flow "tạo đơn hàng lúc checkout" đã tồn tại ở base ([2-base-checkout-create-order-sqd.md](2-base-checkout-create-order-sqd.md)) nay thay đổi lớn: ghi `orders` (items_json) và `order_items` phải nằm trong cùng 1 transaction DB, và phải chống double-submit bằng idempotency key theo `cart_id` + `checkout_attempt_id` khi user bấm checkout 2 lần gần như đồng thời.

```mermaid
sequenceDiagram
    actor Customer
    participant Checkout as Checkout Service
    participant OrderSvc as Order Service
    participant DB as Orders + Order_Items Tables

    par Double-submit, 2 requests gần như đồng thời
        Customer->>Checkout: Submit checkout (request 1, cart_id, attempt_id)
    and
        Customer->>Checkout: Submit checkout (request 2, cart_id, same attempt_id)
    end

    Checkout->>OrderSvc: Create order, idempotency_key = cart_id + attempt_id

    alt Request đầu tiên tới trước, chưa có idempotency_key
        OrderSvc->>DB: BEGIN TRANSACTION
        OrderSvc->>DB: INSERT INTO orders (items_json, idempotency_key, total_amount)
        OrderSvc->>DB: INSERT INTO order_items (order_id, product_id, quantity, unit_price) for each item
        OrderSvc->>DB: COMMIT
        DB-->>OrderSvc: order_id created
        OrderSvc-->>Checkout: Order created
    else Request thứ hai, idempotency_key đã tồn tại
        OrderSvc->>DB: Look up existing order by idempotency_key
        DB-->>OrderSvc: Existing order_id found
        OrderSvc-->>Checkout: Return existing order result, no new order created
    end

    Checkout-->>Customer: Checkout success, order confirmation
```
