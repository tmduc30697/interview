# ERD — Enhance (sau khi thêm cột cents)

Đây là **enhance**: ERD base cộng với cột `amount_cents`/`balance_cents` dual-write song song cột `float` cũ, checkpoint backfill, báo cáo validate từng batch, và feature flag cho phép rollback đọc ngay lập tức. So với base, `ACCOUNT` và `TRANSACTION_` không đổi cấu trúc bảng gốc nhưng có thêm cột `_cents`, và `READ_SOURCE_FLAG` là entity hoàn toàn mới quyết định hệ thống đang đọc từ cột nào.

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ TRANSACTION_ : "records into"
    ACCOUNT ||--o{ BALANCE_VALIDATION_REPORT : "validated by"
    MIGRATION_BATCH ||--o{ BALANCE_VALIDATION_REPORT : produces

    USER {
        string user_id PK
        string full_name
        string email
    }

    ACCOUNT {
        string account_id PK
        string user_id FK
        float balance_float
        int balance_cents
    }

    TRANSACTION_ {
        string transaction_id PK
        string account_id FK
        float amount_float
        int amount_cents
        string type
        datetime created_at
    }

    MIGRATION_BATCH {
        string batch_id PK
        string last_transaction_id_processed
        int batch_size
        string status
        datetime updated_at
    }

    BALANCE_VALIDATION_REPORT {
        string report_id PK
        string batch_id FK
        string account_id FK
        decimal float_balance_as_cents
        int cents_balance
        boolean within_tolerance
        datetime checked_at
    }

    READ_SOURCE_FLAG {
        string flag_id PK
        string flag_name
        string current_value
        datetime updated_at
    }
```
