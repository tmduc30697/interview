# Chuyển tiền nội bộ giữa các tài khoản ngân hàng số

**Hệ thống:** Một app ngân hàng số cho phép user chuyển tiền giữa các tài khoản của chính họ và tài khoản người khác trong cùng hệ thống.

**Vai trò của flow:** Transaction cập nhật số dư của 2 tài khoản (trừ bên gửi, cộng bên nhận) phải lock đúng thứ tự để tránh deadlock khi nhiều lệnh chuyển tiền ngược hướng nhau chạy song song.

**Yêu cầu cụ thể:**
- Mô tả cụ thể kịch bản: transaction A chuyển từ account 1 → account 2, transaction B chuyển từ account 2 → account 1 chạy đồng thời; nếu mỗi transaction lock theo thứ tự "tài khoản nguồn trước, tài khoản đích sau" thì sẽ deadlock — yêu cầu code phải lock theo thứ tự cố định (ví dụ theo account_id tăng dần) để loại trừ deadlock này.
- Chọn isolation level `READ COMMITTED` hoặc cao hơn cho transaction cập nhật số dư, giải thích vì sao `REPEATABLE READ` mặc định của một số DB (MySQL/InnoDB) có thể gây phạm vi lock rộng hơn cần thiết và tăng khả năng deadlock trên các dòng không liên quan.
- Khi DB trả lỗi deadlock (ví dụ mã lỗi 40P01 ở Postgres, 1213 ở MySQL), transaction bị rollback phải được retry tự động tối đa 3 lần với backoff, không được để lỗi văng lên user như một giao dịch thất bại vĩnh viễn.
- Đảm bảo tổng số dư hai tài khoản trước và sau giao dịch không đổi (invariant) dù có retry hay không — viết test giả lập 50 giao dịch ngược hướng chạy song song và assert invariant này.
- Ghi log đầy đủ thông tin deadlock (transaction nào, lock gì, thời điểm) để phục vụ debug, không chỉ log "transaction failed".

---

# Base ERD — Chuyển tiền nội bộ chưa có cơ chế chống deadlock

Đây là **base**: trạng thái hệ thống ngân hàng số *trước khi* áp dụng thứ tự lock cố định, retry và logging deadlock. Suy luận từ đề bài, base đã có `USER` sở hữu nhiều `ACCOUNT`, và mỗi lệnh chuyển tiền tạo 1 `TRANSFER_TRANSACTION` tham chiếu tới account nguồn và account đích. Base chưa có bất kỳ cơ chế nào để tránh deadlock khi 2 giao dịch ngược hướng chạy song song, cũng chưa ghi log chi tiết khi transaction bị lỗi.

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ TRANSFER_TRANSACTION : "là account nguồn"
    ACCOUNT ||--o{ TRANSFER_TRANSACTION : "là account đích"

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
        datetime created_at
    }
```

---

# Base sequence — Transfer money (lock theo thứ tự nguồn trước, đích sau, dẫn tới deadlock)

Đây là **base**, flow chuyển tiền ở trạng thái hiện tại: mỗi transaction lock account nguồn trước rồi mới lock account đích, theo đúng thứ tự tham số truyền vào chứ không chuẩn hóa. Đây chính là kịch bản gây deadlock kinh điển nêu ở yêu cầu 1 của đề bài, khi transaction A chuyển 1 → 2 và transaction B chuyển 2 → 1 chạy đồng thời.

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    participant DB as Database
    participant Acc1 as ACCOUNT 1
    participant Acc2 as ACCOUNT 2

    UserA->>DB: BEGIN Transaction A (chuyển 1 → 2)
    UserB->>DB: BEGIN Transaction B (chuyển 2 → 1)

    DB->>Acc1: Transaction A: SELECT ... FOR UPDATE account 1 (nguồn)
    Acc1-->>DB: Lock granted cho Transaction A

    DB->>Acc2: Transaction B: SELECT ... FOR UPDATE account 2 (nguồn)
    Acc2-->>DB: Lock granted cho Transaction B

    DB->>Acc2: Transaction A: SELECT ... FOR UPDATE account 2 (đích) - chờ lock
    Note over Acc2: Account 2 đang bị Transaction B giữ lock

    DB->>Acc1: Transaction B: SELECT ... FOR UPDATE account 1 (đích) - chờ lock
    Note over Acc1: Account 1 đang bị Transaction A giữ lock

    Note over DB,Acc2: Transaction A chờ Transaction B, Transaction B chờ Transaction A, DEADLOCK
    DB-->>UserA: Lỗi deadlock, transaction bị rollback, không retry
    DB-->>UserB: Transaction còn lại tiếp tục và commit thành công
    DB-->>UserA: Trả lỗi "giao dịch thất bại" thẳng cho user, không log chi tiết
```

---

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

---

# Enhance sequence — Transfer money (lock theo account_id tăng dần, loại trừ deadlock)

Đây là **enhance** của flow `transfer-money` đã có ở base. So với base (lock theo thứ tự nguồn trước - đích sau gây deadlock), nay cả transaction A và B đều chuẩn hóa thứ tự lock theo `account_id` tăng dần trước khi lock, bất kể chiều chuyển tiền, đồng thời dùng isolation level `READ COMMITTED` tường minh. Đáp ứng yêu cầu 1 và 2 của đề bài.

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    participant DB as Database (isolation=READ COMMITTED)
    participant Acc1 as ACCOUNT 1
    participant Acc2 as ACCOUNT 2

    UserA->>DB: BEGIN Transaction A (chuyển 1 → 2)
    UserA->>DB: Chuẩn hóa lock_order = sort([1,2]) = [1, 2]
    UserB->>DB: BEGIN Transaction B (chuyển 2 → 1)
    UserB->>DB: Chuẩn hóa lock_order = sort([2,1]) = [1, 2]

    DB->>Acc1: Transaction A: SELECT ... FOR UPDATE account 1 (theo lock_order)
    Acc1-->>DB: Lock granted cho Transaction A

    DB->>Acc1: Transaction B: SELECT ... FOR UPDATE account 1 (theo lock_order) - chờ lock
    Note over Acc1: Transaction B chờ Transaction A giải phóng account 1, không lock trước account 2

    DB->>Acc2: Transaction A: SELECT ... FOR UPDATE account 2 (theo lock_order)
    Acc2-->>DB: Lock granted cho Transaction A
    DB->>DB: Transaction A: trừ account 1, cộng account 2, COMMIT
    DB->>Acc1: Giải phóng lock account 1

    DB->>Acc1: Transaction B: nhận lock account 1 (vừa được giải phóng)
    Acc1-->>DB: Lock granted cho Transaction B
    DB->>Acc2: Transaction B: SELECT ... FOR UPDATE account 2 (theo lock_order)
    Acc2-->>DB: Lock granted cho Transaction B
    DB->>DB: Transaction B: trừ account 2, cộng account 1, COMMIT

    Note over DB,Acc2: Vì cả 2 transaction đều lock theo cùng thứ tự [1, 2], không bao giờ xảy ra vòng chờ chéo, deadlock được loại trừ hoàn toàn
```

---

# Enhance sequence — Retry tự động và log chi tiết khi vẫn xảy ra deadlock

Đây là **enhance**, flow mới bổ sung cho trường hợp deadlock vẫn xảy ra dù đã chuẩn hóa thứ tự lock (ví dụ do các transaction khác cạnh tranh lock ngoài dự kiến, hoặc index lock phụ). Đáp ứng yêu cầu 3 (retry tự động tối đa 3 lần với backoff, không để lỗi văng lên user) và yêu cầu 5 (ghi log đầy đủ transaction nào, lock gì, thời điểm) của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Transfer Service
    participant DB as Database
    participant Log as DEADLOCK_LOG

    User->>App: Yêu cầu chuyển tiền
    App->>DB: BEGIN Transaction (retry_attempt=1)
    DB-->>App: Lỗi deadlock (mã 40P01/1213), transaction bị rollback tự động

    App->>Log: Ghi DEADLOCK_LOG (transaction_id, blocking_transaction_id, locked_resource, db_error_code, retry_attempt=1, occurred_at)
    App->>App: Backoff (vd 100ms) rồi retry
    App->>DB: BEGIN Transaction (retry_attempt=2)
    DB-->>App: Lỗi deadlock lần nữa

    App->>Log: Ghi DEADLOCK_LOG (retry_attempt=2)
    App->>App: Backoff (vd 200ms) rồi retry
    App->>DB: BEGIN Transaction (retry_attempt=3)
    DB-->>App: Transaction thành công, COMMIT

    App-->>User: Chuyển tiền thành công, user không hề thấy lỗi deadlock trung gian

    Note over App,Log: Nếu cả 3 lần retry đều deadlock, App mới trả lỗi thật cho user, kèm toàn bộ lịch sử retry đã được ghi trong DEADLOCK_LOG để debug
```

---

# Enhance sequence — Test giả lập 50 giao dịch ngược hướng, assert invariant tổng số dư

Đây là **enhance**, flow mới mô tả kịch bản test tự động hoá. Đáp ứng yêu cầu 4 của đề bài: viết test giả lập 50 giao dịch chuyển tiền ngược hướng chạy song song giữa 2 account, rồi assert rằng tổng số dư 2 account trước và sau không đổi, dù có transaction phải retry hay không.

```mermaid
sequenceDiagram
    participant Test as Test Harness
    participant Pool as 50 Transfer Transactions (25 chiều 1→2, 25 chiều 2→1)
    participant DB as Database (lock order + retry đã áp dụng)
    participant Acc1 as ACCOUNT 1
    participant Acc2 as ACCOUNT 2

    Test->>Acc1: Đọc balance ban đầu account 1
    Test->>Acc2: Đọc balance ban đầu account 2
    Test->>Test: Tính total_before = balance1 + balance2

    Test->>Pool: Khởi chạy đồng thời 50 transaction chuyển tiền ngược hướng
    par Chạy song song
        Pool->>DB: Transaction chiều 1→2 (có thể retry tối đa 3 lần)
    and
        Pool->>DB: Transaction chiều 2→1 (có thể retry tối đa 3 lần)
    end
    DB-->>Pool: Toàn bộ 50 transaction cuối cùng đều commit thành công (sau retry nếu cần)

    Test->>Acc1: Đọc balance sau cùng account 1
    Test->>Acc2: Đọc balance sau cùng account 2
    Test->>Test: Tính total_after = balance1 + balance2
    Test->>Test: assert total_after == total_before

    Note over Test,DB: Invariant phải đúng bất kể có bao nhiêu transaction phải retry do deadlock, vì retry chỉ làm chậm chứ không làm sai lệch số tiền chuyển
```
