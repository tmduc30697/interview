# Sequence Diagram — Base: Change Plan

Đây là **base**, flow "đổi gói" gọi Billing và Entitlement riêng lẻ — tiền đề cho saga. Ở base, flow chưa đảm bảo thứ tự chặt chẽ hay khả năng compensate: nếu Entitlement Service lỗi sau khi Billing đã charge thành công, khách có thể bị tính tiền plan mới nhưng chưa được mở tính năng tương ứng.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Subscription Service
    participant Billing as Billing Service
    participant Entitlement as Entitlement Service
    participant Notify as Notification Service

    Customer->>App: Request plan change (new_plan_id)
    App->>Billing: Charge/pro-rate for new plan
    Billing-->>App: Billing succeeded

    App->>Entitlement: Update entitlement to new plan's limits
    Entitlement-->>App: Entitlement updated

    App->>Notify: Send "plan changed" email
    Notify-->>App: Email sent

    App->>App: Update SUBSCRIPTION.plan_id
    App-->>Customer: Plan change confirmed

    Note over App,Entitlement: If entitlement update fails here, billing has already
    Note over App,Entitlement: been charged with no automatic correction
```
