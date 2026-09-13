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
