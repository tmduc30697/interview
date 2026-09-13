# Sequence Diagram — Enhance: Place Order (Happy Path)

Đây là **enhance**, flow "đặt hàng" đã có ở base ([2-base-place-order-sqd.md](2-base-place-order-sqd.md)) nhưng nay thay đổi căn bản: một Saga Orchestrator điều phối từng bước thay vì Order Service gọi trực tiếp, mỗi bước được ghi thành `SAGA_STEP` với `SAGA_EVENT` tương ứng, phục vụ replay/audit sau này.

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
