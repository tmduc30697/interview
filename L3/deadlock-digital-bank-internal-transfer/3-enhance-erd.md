# Enhance ERD — Thêm thứ tự lock cố định, retry và deadlock logging

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `TRANSFER_TRANSACTION` thêm các trường theo dõi isolation level và số lần retry, và có thêm entity mới `DEADLOCK_LOG` để ghi chi tiết mỗi lần deadlock xảy ra, ứng trực tiếp với các yêu cầu:

- `TRANSFER_TRANSACTION.isolation_level` và `lock_order_account_ids` — đáp ứng yêu cầu 1 (lock theo account_id tăng dần) và yêu cầu 2 (chọn READ COMMITTED tường minh thay vì mặc định REPEATABLE READ).
- `TRANSFER_TRANSACTION.retry_count`/`max_retry` — đáp ứng yêu cầu 3 (retry tự động tối đa 3 lần khi gặp lỗi deadlock).
- `DEADLOCK_LOG` (mới) — đáp ứng yêu cầu 5 (log đầy đủ transaction nào, lock gì, thời điểm), dữ liệu này cũng phục vụ test invariant ở yêu cầu 4.

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ TRANSFER_TRANSACTION : "là account nguồn"
    ACCOUNT ||--o{ TRANSFER_TRANSACTION : "là account đích"
    TRANSFER_TRANSACTION ||--o{ DEADLOCK_LOG : "có thể phát sinh"

    USER {
        string id PK
        string name
    }
    ACCOUNT {
        string id PK
        string user_id FK
        decimal balance
    }
    TRANSFER_TRANSACTION {
        string id PK
        string from_account_id FK
        string to_account_id FK
        decimal amount
        string status "pending|success|failed"
        string isolation_level "READ_COMMITTED"
        string lock_order_account_ids "vd: [1,2] đã sort tăng dần"
        int retry_count
        int max_retry "3"
        datetime created_at
    }
    DEADLOCK_LOG {
        string id PK
        string transaction_id FK
        string blocking_transaction_id
        string locked_resource "vd: account_id=2 row lock"
        string db_error_code "40P01 | 1213"
        int retry_attempt
        datetime occurred_at
    }
```
