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
