# ERD — Enhance (sau khi có đối soát billing với processor)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — khớp nhóm invoice proration với đúng một lượt thu, phân biệt "đang retry" với "thất bại hẳn" ở dunning, cập nhật lại đối soát khi có hoàn tiền sau khớp, tách phí xử lý khỏi số tiền gộp, và xử lý riêng giao dịch cận ranh giới chu kỳ. So với base, `PROCESSOR_CHARGE` nay lưu thêm `processing_fee` và `net_amount`, và có thêm `DUNNING_ATTEMPT` để theo dõi lịch sử retry thay vì chỉ một status tĩnh.

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : has
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--o{ INVOICE : generates
    INVOICE }o--o| PROCESSOR_CHARGE : "collected via"
    PROCESSOR_CHARGE ||--o{ DUNNING_ATTEMPT : "may retry via"

    INVOICE ||--o| REFUND_EVENT : "may have"
    REFUND_EVENT |o--o| RECONCILIATION_DISCREPANCY : "re-opens"

    RECONCILIATION_RUN ||--o{ RECONCILIATION_MATCH : produces
    RECONCILIATION_MATCH ||--o{ INVOICE : groups
    RECONCILIATION_MATCH |o--o| PROCESSOR_CHARGE : "matches (optional)"
    RECONCILIATION_MATCH ||--o| RECONCILIATION_DISCREPANCY : "may yield"

    CUSTOMER {
        string customer_id PK
        string company_name
        string email
    }

    PLAN {
        string plan_id PK
        string name
        decimal monthly_price
    }

    SUBSCRIPTION {
        string subscription_id PK
        string customer_id FK
        string plan_id FK
        string status
        date current_period_start
        date current_period_end
    }

    INVOICE {
        string invoice_id PK
        string subscription_id FK
        string type
        decimal amount
        string currency
        string status
        datetime created_at
    }

    PROCESSOR_CHARGE {
        string charge_id PK
        string processor_reference_id
        decimal gross_amount
        decimal processing_fee
        decimal net_amount
        string status
        datetime charged_at
        datetime settled_at
    }

    DUNNING_ATTEMPT {
        string attempt_id PK
        string charge_id FK
        int attempt_number
        string result
        datetime attempted_at
        datetime next_retry_at
    }

    REFUND_EVENT {
        string refund_event_id PK
        string invoice_id FK
        decimal amount
        datetime occurred_at
    }

    RECONCILIATION_RUN {
        string run_id PK
        date cycle_period_start
        date cycle_period_end
        string status
    }

    RECONCILIATION_MATCH {
        string match_id PK
        string run_id FK
        string charge_id FK
        decimal invoice_total
        decimal charge_net_amount
        string match_status
    }

    RECONCILIATION_DISCREPANCY {
        string discrepancy_id PK
        string match_id FK
        string type
        decimal amount_diff
        string status
    }
```
