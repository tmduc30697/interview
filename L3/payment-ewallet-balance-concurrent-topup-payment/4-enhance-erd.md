# Enhance ERD — Thêm idempotency, ledger đối chiếu và cảnh báo lệch số dư

Đây là **enhance**, mô hình dữ liệu sau khi áp cơ chế update nguyên tử có điều kiện lên base. So với base, `TOPUP_TRANSACTION` thêm `bank_reference_id` unique để idempotent theo mã giao dịch ngân hàng khi callback đến trùng, `WALLET` thêm `status` để có thể tạm khóa khi phát hiện lệch số dư. Thêm mới `WALLET_LEDGER_ENTRY` — ghi bất biến mọi lần cộng/trừ số dư, làm nguồn để job đối chiếu định kỳ tính tổng và so sánh với `WALLET.balance`; và `BALANCE_RECONCILIATION_ALERT` — ghi nhận lần lệch số dư phát hiện được, kèm trạng thái điều tra.

```mermaid
erDiagram
    USER ||--|| WALLET : owns
    USER ||--o{ BANK_ACCOUNT : links
    WALLET ||--o{ TOPUP_TRANSACTION : "credited by"
    WALLET ||--o{ PAYMENT_TRANSACTION : "debited by"
    WALLET ||--o{ WALLET_LEDGER_ENTRY : records
    WALLET ||--o{ BALANCE_RECONCILIATION_ALERT : "flagged by"
    MERCHANT ||--o{ PAYMENT_TRANSACTION : receives

    USER {
        string id PK
        string name
    }
    WALLET {
        string id PK
        string user_id FK
        decimal balance
        string status "active | locked_for_investigation"
    }
    BANK_ACCOUNT {
        string id PK
        string user_id FK
        string bank_name
        string account_number
    }
    TOPUP_TRANSACTION {
        string id PK
        string wallet_id FK
        string bank_account_id FK
        string bank_reference_id UK "idempotency key, dùng để phát hiện callback trùng"
        decimal amount
        string status "success | duplicate_ignored"
        datetime created_at
    }
    PAYMENT_TRANSACTION {
        string id PK
        string wallet_id FK
        string merchant_id FK
        decimal amount
        string status "success | rejected_insufficient_balance"
        datetime created_at
    }
    MERCHANT {
        string id PK
        string name
    }
    WALLET_LEDGER_ENTRY {
        string id PK
        string wallet_id FK
        string source_type "topup | payment"
        string source_id "TOPUP_TRANSACTION.id hoặc PAYMENT_TRANSACTION.id"
        decimal delta_amount "dương khi cộng, âm khi trừ"
        decimal resulting_balance "số dư sau khi entry này được ghi atomic"
        datetime created_at
    }
    BALANCE_RECONCILIATION_ALERT {
        string id PK
        string wallet_id FK
        decimal expected_balance "tổng cộng dồn từ WALLET_LEDGER_ENTRY"
        decimal actual_balance "WALLET.balance tại thời điểm đối chiếu"
        string status "open | resolved"
        datetime detected_at
    }
```
