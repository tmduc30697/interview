# Sequence Diagram — Base: Place Order

Đây là **base**, flow "đặt hàng" gọi tuần tự 3 service (Inventory, Payment, Shipping) — tiền đề cho saga: chính chuỗi bước reserve inventory → charge payment → create shipment ở đây là những gì saga sẽ điều phối lại. Ở base, flow gọi trực tiếp không qua orchestrator, không có compensate, không có persisted state, nên nếu một bước giữa chừng thất bại thì các bước trước đó không tự động được hoàn tác.

```mermaid
sequenceDiagram
    actor Customer
    participant Order as Order Service
    participant Inventory as Inventory Service
    participant Payment as Payment Service
    participant Shipping as Shipping Service

    Customer->>Order: Checkout, confirm order
    Order->>Order: Create ORDER (status=pending)

    Order->>Inventory: Reserve stock for each ORDER_ITEM
    Inventory-->>Order: Stock reserved

    Order->>Payment: Charge customer total_amount
    Payment-->>Order: Payment captured

    Order->>Shipping: Create shipment for order
    Shipping-->>Order: Shipment created

    Order->>Order: Mark ORDER status=confirmed
    Order-->>Customer: Order confirmed

    Note over Order,Shipping: If payment or shipping fails partway, nothing here
    Note over Order,Shipping: automatically releases the inventory already reserved
```
