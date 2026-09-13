# Sequence Diagram — Enhance: Change Plan (Happy Path)

Đây là **enhance**, flow "đổi gói" đã có ở base ([2-base-change-plan-sqd.md](2-base-change-plan-sqd.md)) nhưng nay thay đổi căn bản: thứ tự saga chặt chẽ (billing trước, entitlement chỉ mở sau khi billing xác nhận thành công), và notification chỉ gửi sau khi cả hai đã ổn định, thay vì gọi rời rạc như base.

```mermaid
sequenceDiagram
    actor Customer
    participant Orchestrator as Saga Orchestrator
    participant Billing as Billing Service
    participant Entitlement as Entitlement Service
    participant Notify as Notification Service

    Customer->>Orchestrator: Request plan change (new_plan_id)
    Orchestrator->>Orchestrator: Create PLAN_CHANGE_HISTORY, create SAGA_INSTANCE

    Orchestrator->>Billing: Step 1, charge/pro-rate for new plan
    Billing-->>Orchestrator: Billing succeeded
    Orchestrator->>Orchestrator: Create SAGA_STEP (billing_charged, status=completed)

    Orchestrator->>Entitlement: Step 2, only now update entitlement to new plan's limits
    Entitlement-->>Orchestrator: Entitlement updated
    Orchestrator->>Orchestrator: Create SAGA_STEP (entitlement_updated, status=completed)

    Orchestrator->>Orchestrator: Mark SAGA_INSTANCE status=completed
    Orchestrator->>Notify: Step 3, only now send "plan changed" email
    Notify-->>Orchestrator: Email sent

    Orchestrator->>Orchestrator: Update PLAN_CHANGE_HISTORY (result=success, resolved_at)
    Orchestrator-->>Customer: Plan change confirmed
```
