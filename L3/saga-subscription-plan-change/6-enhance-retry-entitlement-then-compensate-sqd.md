# Sequence Diagram — Enhance: Retry Entitlement Then Compensate

Đây là **enhance**, flow mới xử lý trường hợp Entitlement Service down đúng lúc cần cập nhật sau khi billing đã thành công. Flow này thể hiện yêu cầu retry với backoff trong khoảng thời gian giới hạn, và nếu vẫn thất bại thì compensate hoàn tiền phần chênh lệch, giữ nguyên plan cũ.

```mermaid
sequenceDiagram
    participant Orchestrator as Saga Orchestrator
    participant Billing as Billing Service
    participant Entitlement as Entitlement Service
    actor Customer

    Orchestrator->>Billing: Step 1, charge/pro-rate for new plan
    Billing-->>Orchestrator: Billing succeeded
    Orchestrator->>Orchestrator: Create SAGA_STEP (billing_charged, status=completed)

    Orchestrator->>Entitlement: Step 2, update entitlement
    Entitlement-->>Orchestrator: Service unavailable

    Orchestrator->>Orchestrator: Create COMPENSATION_ATTEMPT (attempt_number=1)
    Orchestrator->>Orchestrator: Schedule retry with backoff

    loop retry within limited time window
        Orchestrator->>Entitlement: Retry update entitlement
        Entitlement-->>Orchestrator: Still unavailable
        Orchestrator->>Orchestrator: Create COMPENSATION_ATTEMPT (attempt_number+1)
    end

    Orchestrator->>Orchestrator: Time window exceeded, give up on entitlement update
    Orchestrator->>Billing: Compensate, refund the pro-rated charge difference
    Billing-->>Orchestrator: Refund issued

    Orchestrator->>Orchestrator: Update SUBSCRIPTION, keep old plan_id
    Orchestrator->>Orchestrator: Update PLAN_CHANGE_HISTORY (result=failed_compensated)
    Orchestrator-->>Customer: Plan change failed, refunded, still on your previous plan
```
