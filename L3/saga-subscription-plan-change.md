# Nâng/hạ cấp gói subscription xuyên nhiều service (billing + entitlement + notification)

**Hệ thống:** SaaS B2B cho phép khách hàng nâng cấp/hạ cấp plan, việc này phải đồng bộ giữa billing (tính tiền), entitlement (mở/khóa tính năng), và notification (báo khách).

**Vai trò của flow:** Saga đảm bảo billing và entitlement luôn khớp nhau — không có trường hợp khách bị charge tiền plan mới nhưng chưa được mở tính năng, hoặc ngược lại.

**Yêu cầu cụ thể:**
- Thứ tự saga: charge/pro-rate billing trước, chỉ mở entitlement sau khi billing xác nhận thành công; nếu billing thất bại, không chạm tới entitlement.
- Nếu entitlement service down đúng lúc cần cập nhật sau khi billing đã thành công, saga phải retry với backoff trong một khoảng thời gian giới hạn, và nếu vẫn thất bại phải compensate (hoàn tiền phần chênh lệch) và giữ plan cũ.
- Downgrade plan phải xử lý được trường hợp khách đang dùng tính năng vượt hạn mức plan mới (ví dụ vượt số user) — quyết định rõ: chặn downgrade hay downgrade kèm cảnh báo.
- Notification chỉ được gửi sau khi cả billing và entitlement đã ổn định (không gửi email "đã nâng cấp thành công" rồi sau đó saga phải rollback).
- Toàn bộ lịch sử thay đổi plan của một khách phải truy vấn lại được (ai đổi plan gì, khi nào, kết quả ra sao) để phục vụ hỗ trợ khách hàng.

---

# ERD — Base (trước khi có saga đổi gói)

Đây là **base**: mô hình dữ liệu suy luận cho SaaS B2B *trước khi* có saga điều phối đổi gói. Đề bài giả định hệ thống đã có billing, entitlement, và notification như 3 service riêng biệt, mỗi khách hàng có subscription và entitlement tương ứng gói hiện tại — nếu không có sẵn các entity này thì "saga đảm bảo billing và entitlement luôn khớp nhau" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ đổi gói, không suy diễn thêm các module không liên quan (support, usage analytics...).

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : has
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--|| ENTITLEMENT : grants
    SUBSCRIPTION ||--o{ BILLING_INVOICE : generates

    CUSTOMER {
        string customer_id PK
        string company_name
        string email
    }

    PLAN {
        string plan_id PK
        string name
        decimal monthly_price
        string feature_limits
    }

    SUBSCRIPTION {
        string subscription_id PK
        string customer_id FK
        string plan_id FK
        string status
    }

    ENTITLEMENT {
        string entitlement_id PK
        string subscription_id FK
        string enabled_features
        int user_limit
        string status
    }

    BILLING_INVOICE {
        string invoice_id PK
        string subscription_id FK
        decimal amount
        string status
        datetime created_at
    }
```

---

<a id="2-base-change-plan-sqd"></a>

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

---

# ERD — Enhance (sau khi có saga đổi gói)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — thứ tự saga rõ ràng (billing trước, entitlement sau), retry/compensate khi entitlement down, kiểm tra hạn mức khi downgrade, notification chỉ gửi sau khi ổn định, và lịch sử đổi gói truy vấn được. So với base, `SUBSCRIPTION` nay được dẫn dắt bởi `SAGA_INSTANCE`, và có thêm `PLAN_CHANGE_HISTORY` ghi lại toàn bộ lượt đổi gói.

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : has
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--|| ENTITLEMENT : grants
    SUBSCRIPTION ||--o{ BILLING_INVOICE : generates

    SUBSCRIPTION ||--o{ PLAN_CHANGE_HISTORY : records
    PLAN_CHANGE_HISTORY ||--|| SAGA_INSTANCE : "tracked by"
    SAGA_INSTANCE ||--o{ SAGA_STEP : consists_of
    SAGA_STEP |o--o| COMPENSATION_ATTEMPT : "may trigger"

    CUSTOMER {
        string customer_id PK
        string company_name
        string email
    }

    PLAN {
        string plan_id PK
        string name
        decimal monthly_price
        string feature_limits
        int user_limit
    }

    SUBSCRIPTION {
        string subscription_id PK
        string customer_id FK
        string plan_id FK
        string status
    }

    ENTITLEMENT {
        string entitlement_id PK
        string subscription_id FK
        string enabled_features
        int user_limit
        string status
    }

    BILLING_INVOICE {
        string invoice_id PK
        string subscription_id FK
        decimal amount
        string status
        datetime created_at
    }

    PLAN_CHANGE_HISTORY {
        string change_id PK
        string subscription_id FK
        string from_plan_id
        string to_plan_id
        string requested_by
        string result
        datetime requested_at
        datetime resolved_at
    }

    SAGA_INSTANCE {
        string saga_id PK
        string change_id FK
        string current_step
        string status
    }

    SAGA_STEP {
        string step_id PK
        string saga_id FK
        string name
        string status
        datetime executed_at
    }

    COMPENSATION_ATTEMPT {
        string attempt_id PK
        string step_id FK
        int attempt_number
        string status
        datetime next_retry_at
    }
```

---

# Sequence Diagram — Enhance: Change Plan (Happy Path)

Đây là **enhance**, flow "đổi gói" đã có ở base ([2-base-change-plan-sqd.md](#2-base-change-plan-sqd)) nhưng nay thay đổi căn bản: thứ tự saga chặt chẽ (billing trước, entitlement chỉ mở sau khi billing xác nhận thành công), và notification chỉ gửi sau khi cả hai đã ổn định, thay vì gọi rời rạc như base.

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

---

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

---

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
