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
