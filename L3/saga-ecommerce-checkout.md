# Checkout e-commerce nhiều bước (payment + inventory + shipping)

**Hệ thống:** Trang e-commerce với luồng đặt hàng chạm tới 3 service riêng: Payment, Inventory, Shipping.

**Vai trò của flow:** Saga orchestration điều phối toàn bộ transaction nghiệp vụ xuyên service, và compensate (hủy ngược) khi một bước giữa đường thất bại.

**Yêu cầu cụ thể:**
- Định nghĩa rõ từng bước saga (reserve inventory → charge payment → create shipment) và compensating action tương ứng cho mỗi bước (release inventory, refund payment, cancel shipment).
- Nếu bước charge payment thất bại sau khi đã reserve inventory thành công, phải tự động gọi compensate để release inventory, và toàn bộ order chuyển trạng thái "failed" chứ không kẹt ở trạng thái lửng lơ.
- Saga state (đang ở bước nào, đã compensate bước nào) phải được persist, để nếu orchestrator service crash giữa saga, có thể resume đúng chỗ khi restart.
- Xử lý được trường hợp compensate cũng thất bại (ví dụ refund payment lỗi) — phải có retry với backoff và escalate sang xử lý thủ công/alert nếu vượt số lần retry.
- Toàn bộ event của saga (bắt đầu, mỗi step, compensate) phải được lưu dưới dạng event log để có thể replay/audit lại toàn bộ hành trình của một order.

---

# ERD — Base (trước khi có saga orchestration)

Đây là **base**: mô hình dữ liệu suy luận cho checkout e-commerce *trước khi* có saga điều phối. Đề bài giả định hệ thống đã có `ORDER`, đã tích hợp Inventory/Payment/Shipping như 3 service riêng biệt — nếu không có sẵn các entity này thì "saga điều phối transaction xuyên 3 service" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ checkout, không suy diễn thêm các module không liên quan (khuyến mãi, đánh giá sản phẩm...).

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    ORDER_ITEM }o--|| INVENTORY_ITEM : reserves
    ORDER ||--o| PAYMENT_CHARGE : "paid via"
    ORDER ||--o| SHIPMENT : "shipped via"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        decimal total_amount
        string status
        datetime created_at
    }

    ORDER_ITEM {
        string order_item_id PK
        string order_id FK
        string sku
        int quantity
    }

    INVENTORY_ITEM {
        string sku PK
        int available_quantity
        int reserved_quantity
    }

    PAYMENT_CHARGE {
        string charge_id PK
        string order_id FK
        decimal amount
        string status
    }

    SHIPMENT {
        string shipment_id PK
        string order_id FK
        string status
        string tracking_code
    }
```

---

<a id="2-base-place-order-sqd"></a>

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

---

# ERD — Enhance (sau khi có saga orchestration)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — định nghĩa từng bước saga và compensate tương ứng, persist saga state để resume khi crash, retry/escalate khi compensate thất bại, và event log để replay/audit. So với base, `ORDER` không đổi cấu trúc nhưng nay trạng thái của nó được dẫn dắt bởi `SAGA_INSTANCE` thay vì được set trực tiếp từ Order Service.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    ORDER_ITEM }o--|| INVENTORY_ITEM : reserves
    ORDER ||--o| PAYMENT_CHARGE : "paid via"
    ORDER ||--o| SHIPMENT : "shipped via"

    ORDER ||--|| SAGA_INSTANCE : "orchestrated by"
    SAGA_INSTANCE ||--o{ SAGA_STEP : consists_of
    SAGA_STEP ||--o{ SAGA_EVENT : logs
    SAGA_STEP |o--o| COMPENSATION_ATTEMPT : "may trigger"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        decimal total_amount
        string status
        datetime created_at
    }

    ORDER_ITEM {
        string order_item_id PK
        string order_id FK
        string sku
        int quantity
    }

    INVENTORY_ITEM {
        string sku PK
        int available_quantity
        int reserved_quantity
    }

    PAYMENT_CHARGE {
        string charge_id PK
        string order_id FK
        decimal amount
        string status
    }

    SHIPMENT {
        string shipment_id PK
        string order_id FK
        string status
        string tracking_code
    }

    SAGA_INSTANCE {
        string saga_id PK
        string order_id FK
        string current_step
        string status
        datetime started_at
        datetime updated_at
    }

    SAGA_STEP {
        string step_id PK
        string saga_id FK
        string name
        string status
        int sequence_order
        datetime executed_at
    }

    COMPENSATION_ATTEMPT {
        string attempt_id PK
        string step_id FK
        int attempt_number
        string status
        datetime attempted_at
        datetime next_retry_at
    }

    SAGA_EVENT {
        string event_id PK
        string step_id FK
        string event_type
        string payload
        datetime created_at
    }
```

---

# Sequence Diagram — Enhance: Place Order (Happy Path)

Đây là **enhance**, flow "đặt hàng" đã có ở base ([2-base-place-order-sqd.md](#2-base-place-order-sqd)) nhưng nay thay đổi căn bản: một Saga Orchestrator điều phối từng bước thay vì Order Service gọi trực tiếp, mỗi bước được ghi thành `SAGA_STEP` với `SAGA_EVENT` tương ứng, phục vụ replay/audit sau này.

```mermaid
sequenceDiagram
    actor Customer
    participant Orchestrator as Saga Orchestrator
    participant Inventory as Inventory Service
    participant Payment as Payment Service
    participant Shipping as Shipping Service

    Customer->>Orchestrator: Checkout, confirm order
    Orchestrator->>Orchestrator: Create ORDER, create SAGA_INSTANCE (status=running)
    Orchestrator->>Orchestrator: Log SAGA_EVENT (saga started)

    Orchestrator->>Inventory: Step 1, reserve inventory
    Inventory-->>Orchestrator: Stock reserved
    Orchestrator->>Orchestrator: Create SAGA_STEP (reserve_inventory, status=completed), log event

    Orchestrator->>Payment: Step 2, charge payment
    Payment-->>Orchestrator: Payment captured
    Orchestrator->>Orchestrator: Create SAGA_STEP (charge_payment, status=completed), log event

    Orchestrator->>Shipping: Step 3, create shipment
    Shipping-->>Orchestrator: Shipment created
    Orchestrator->>Orchestrator: Create SAGA_STEP (create_shipment, status=completed), log event

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=completed, ORDER status=confirmed
    Orchestrator-->>Customer: Order confirmed
```

---

# Sequence Diagram — Enhance: Compensate Failed Checkout

Đây là **enhance**, flow mới xử lý trường hợp bước charge payment thất bại sau khi đã reserve inventory thành công. Flow này thể hiện yêu cầu "tự động gọi compensate để release inventory, order chuyển trạng thái failed chứ không kẹt lửng lơ" — điều mà base hoàn toàn không có.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator
    participant Inventory as Inventory Service
    participant Payment as Payment Service

    Orchestrator->>Inventory: Step 1, reserve inventory
    Inventory-->>Orchestrator: Stock reserved
    Orchestrator->>Orchestrator: Create SAGA_STEP (reserve_inventory, status=completed)

    Orchestrator->>Payment: Step 2, charge payment
    Payment-->>Orchestrator: Payment failed (card declined)
    Orchestrator->>Orchestrator: Create SAGA_STEP (charge_payment, status=failed), log event

    Orchestrator->>Orchestrator: Determine compensating actions for all completed steps
    Orchestrator->>Inventory: Compensate, release reserved stock
    Inventory-->>Orchestrator: Stock released
    Orchestrator->>Orchestrator: Log SAGA_EVENT (compensated reserve_inventory)

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=compensated, ORDER status=failed
    Orchestrator-->>Orchestrator: No shipment step ever attempted, nothing to compensate there
```

---

# Sequence Diagram — Enhance: Resume Saga After Orchestrator Crash

Đây là **enhance**, flow mới xử lý trường hợp Saga Orchestrator crash giữa chừng một saga. Flow này thể hiện yêu cầu "saga state phải được persist để resume đúng chỗ khi restart" — dựa vào `SAGA_INSTANCE.current_step` đã lưu thay vì chạy lại toàn bộ saga từ đầu.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator (new instance)
    participant Payment as Payment Service
    participant Shipping as Shipping Service

    Note over Orchestrator: Orchestrator process restarted after a crash

    Orchestrator->>Orchestrator: Query all SAGA_INSTANCE with status=running
    Orchestrator->>Orchestrator: Load SAGA_STEP history for this saga_id, find last completed step

    Note over Orchestrator: reserve_inventory was already completed before the crash
    Note over Orchestrator: charge_payment was in progress and not confirmed either way

    Orchestrator->>Payment: Query payment status by order_id (idempotency check)
    Payment-->>Orchestrator: No successful charge found for this order

    Orchestrator->>Payment: Retry step, charge payment
    Payment-->>Orchestrator: Payment captured
    Orchestrator->>Orchestrator: Create SAGA_STEP (charge_payment, status=completed)

    Orchestrator->>Shipping: Continue to step 3, create shipment
    Shipping-->>Orchestrator: Shipment created

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=completed
    Note over Orchestrator: No duplicate inventory reservation, saga resumed exactly where it left off
```

---

# Sequence Diagram — Enhance: Escalate Failed Compensation

Đây là **enhance**, flow mới xử lý trường hợp compensate cũng thất bại (ví dụ refund payment lỗi). Flow này thể hiện yêu cầu "retry với backoff, escalate sang xử lý thủ công/alert nếu vượt số lần retry" — trạng thái mà một saga đơn giản không xử lý được.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator
    participant Payment as Payment Service
    participant Alert as Alert Service
    actor Ops as Ops Team

    Orchestrator->>Orchestrator: Shipping step failed, need to compensate charge_payment
    Orchestrator->>Payment: Compensate, refund payment
    Payment-->>Orchestrator: Refund failed (processor error)

    Orchestrator->>Orchestrator: Create COMPENSATION_ATTEMPT (attempt_number=1, status=failed)
    Orchestrator->>Orchestrator: Schedule retry with backoff (next_retry_at)

    loop retry until max attempts reached
        Orchestrator->>Payment: Retry compensate, refund payment
        Payment-->>Orchestrator: Refund still failing
        Orchestrator->>Orchestrator: Create COMPENSATION_ATTEMPT (attempt_number+1, status=failed)
    end

    Orchestrator->>Orchestrator: Max retry attempts exceeded
    Orchestrator->>Alert: Escalate, compensation stuck for this order
    Alert-->>Ops: Urgent alert, manual refund needed

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=compensation_failed
    Ops->>Payment: Manually process refund outside the saga
    Ops->>Orchestrator: Mark saga resolved after manual intervention
```
