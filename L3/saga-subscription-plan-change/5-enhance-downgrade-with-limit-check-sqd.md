# Sequence Diagram — Enhance: Downgrade With Limit Check

Đây là **enhance**, flow mới xử lý downgrade khi khách đang dùng tính năng vượt hạn mức plan mới (ví dụ vượt số user). Flow này thể hiện quyết định "chặn downgrade" khi vượt hạn mức, thay vì âm thầm cắt tính năng gây gián đoạn dịch vụ.

```mermaid
sequenceDiagram
    actor Customer
    participant Orchestrator as Saga Orchestrator
    participant Entitlement as Entitlement Service
    participant Billing as Billing Service

    Customer->>Orchestrator: Request downgrade to new_plan_id
    Orchestrator->>Entitlement: Check current usage against new plan's limits
    Entitlement-->>Orchestrator: Current user_count exceeds new plan's user_limit

    Orchestrator->>Orchestrator: Create PLAN_CHANGE_HISTORY (result=blocked)
    Orchestrator-->>Customer: Downgrade blocked, reduce user count to N or fewer first

    Note over Orchestrator,Billing: Billing step never runs, saga stops before touching money
```
