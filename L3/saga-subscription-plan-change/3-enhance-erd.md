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
