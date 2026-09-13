# ERD — Enhance (sau khi có đối soát cuối ngày)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — nhận file batch đối tác (có thể trễ/khác múi giờ), phân loại 3 loại sai lệch, xử lý giao dịch bị đối tác báo hoàn sau khi đã khớp, audit trail cho mọi điều chỉnh số dư, và ngưỡng cảnh báo tự động. So với base, `LEDGER_ENTRY` không đổi cấu trúc nhưng nay còn được tạo bởi `ADJUSTMENT_TRANSACTION` (không chỉ bởi `TRANSACTION` gốc) — đây là cách duy nhất để sửa số dư, không sửa trực tiếp.

```mermaid
erDiagram
    CUSTOMER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ CARD : has
    ACCOUNT ||--o{ TRANSACTION : "posts to"
    TRANSACTION ||--|| LEDGER_ENTRY : records
    ACCOUNT ||--o{ ADJUSTMENT_TRANSACTION : "adjusts balance of"
    ADJUSTMENT_TRANSACTION ||--|| LEDGER_ENTRY : "also records via"

    PARTNER_SETTLEMENT_FILE ||--o{ PARTNER_SETTLEMENT_RECORD : contains
    PARTNER_SETTLEMENT_RECORD |o--o| TRANSACTION : "matches (optional)"

    RECONCILIATION_RUN ||--o{ PARTNER_SETTLEMENT_RECORD : processes
    RECONCILIATION_RUN ||--o{ RECONCILIATION_DISCREPANCY : yields
    RECONCILIATION_RUN ||--o| RECONCILIATION_ALERT : "may trigger"
    RECONCILIATION_DISCREPANCY |o--o| ADJUSTMENT_TRANSACTION : "resolved by"

    TRANSACTION |o--o| PARTNER_REVERSAL_NOTICE : "later reversed by"
    PARTNER_REVERSAL_NOTICE |o--o| ADJUSTMENT_TRANSACTION : "resolved by"

    ADJUSTMENT_TRANSACTION ||--|| AUDIT_LOG : "always writes"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
        string kyc_status
    }

    ACCOUNT {
        string account_id PK
        string customer_id FK
        string currency
        decimal balance
        string status
    }

    CARD {
        string card_id PK
        string account_id FK
        string card_network
        string card_number_masked
        string status
    }

    TRANSACTION {
        string transaction_id PK
        string card_id FK
        string account_id FK
        decimal amount
        string currency
        string merchant_name
        string status
        string partner_reference_id
        datetime created_at
    }

    LEDGER_ENTRY {
        string entry_id PK
        string transaction_id FK
        string adjustment_id FK
        string account_id FK
        decimal amount
        string entry_type
        decimal balance_after
        datetime created_at
    }

    PARTNER_SETTLEMENT_FILE {
        string file_id PK
        string partner_name
        date business_date
        string partner_timezone
        datetime received_at
        string status
    }

    PARTNER_SETTLEMENT_RECORD {
        string record_id PK
        string file_id FK
        string partner_transaction_id
        decimal amount
        string currency
        datetime transaction_datetime_partner
        date normalized_business_date
        string matched_transaction_id FK
    }

    RECONCILIATION_RUN {
        string run_id PK
        date business_date
        datetime started_at
        datetime completed_at
        decimal total_discrepancy_amount
        string status
    }

    RECONCILIATION_DISCREPANCY {
        string discrepancy_id PK
        string run_id FK
        string type
        string transaction_id FK
        string record_id FK
        decimal amount_diff
        string priority
        string status
    }

    ADJUSTMENT_TRANSACTION {
        string adjustment_id PK
        string discrepancy_id FK
        string reversal_id FK
        string account_id FK
        decimal amount
        string reason
        string created_by
        datetime created_at
    }

    PARTNER_REVERSAL_NOTICE {
        string reversal_id PK
        string partner_transaction_id
        string original_transaction_id FK
        decimal reversal_amount
        string reason
        datetime received_at
        datetime processed_at
    }

    RECONCILIATION_ALERT {
        string alert_id PK
        string run_id FK
        decimal threshold_amount
        decimal actual_amount
        datetime triggered_at
        string acknowledged_by
        datetime acknowledged_at
    }

    AUDIT_LOG {
        string log_id PK
        string adjustment_id FK
        string action
        string actor
        string before_state
        string after_state
        datetime created_at
    }
```
