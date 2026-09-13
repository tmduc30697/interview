# Đổi đơn vị tiền tệ lưu trữ từ float sang integer (cents) trong hệ thống fintech

**Hệ thống:** Một app quản lý chi tiêu cá nhân đang lưu số tiền dạng `float` (dễ sai số làm tròn), cần migrate toàn bộ sang lưu `integer` (đơn vị cents) mà không làm sai lệch số dư của user nào trong lúc migrate.

**Vai trò của flow:** Flow dual-write phải đảm bảo mọi giao dịch mới trong lúc migrate được ghi đúng ở cả cột `amount_float` (cũ) và `amount_cents` (mới), với công thức chuyển đổi nhất quán, để không tạo ra sai số dồn tích ảnh hưởng đến số dư hiển thị cho user.

**Yêu cầu cụ thể:**
- Quy định công thức chuyển đổi cụ thể (ví dụ round-half-up ở đơn vị cents) và bắt buộc áp dụng đồng nhất ở cả chiều ghi mới và chiều backfill dữ liệu cũ, để tránh 2 cách round khác nhau tạo lệch số dư giữa 2 cột.
- Mô tả race condition: giao dịch A (ghi tiền vào) và giao dịch B (rút tiền ra) xảy ra gần như đồng thời trên cùng 1 tài khoản trong giai đoạn dual-write — yêu cầu cả 2 cột `amount_float` và `amount_cents` phải được cập nhật trong cùng transaction với cùng 1 lock trên dòng số dư, không cho phép trạng thái chỉ 1 cột được update.
- Job backfill lịch sử giao dịch phải tính lại `amount_cents` từ `amount_float` gốc (không phải từ số dư đã tính toán ngược), và phải chạy khi hệ thống ở mức traffic thấp theo batch, có cơ chế tạm dừng nếu phát hiện tỷ lệ lệch số dư vượt ngưỡng.
- Có bước validate: sau khi backfill xong mỗi batch, tổng số dư tính theo cột cũ và cột mới của từng user phải khớp trong sai số cho phép (ví dụ dưới 1 cent do rounding), nếu lệch phải chặn không cho tiếp tục sang batch kế và cảnh báo ngay.
- Định nghĩa rollback plan cụ thể: nếu phát hiện lỗi nghiêm trọng sau khi đã chuyển đọc sang cột mới, phải có cách quay lại đọc cột cũ ngay lập tức (feature flag) mà không cần deploy lại code.

---

# ERD — Base (trước khi đổi sang lưu cents)

Đây là **base**: mô hình dữ liệu suy luận cho app quản lý chi tiêu cá nhân *trước khi* đổi đơn vị lưu trữ tiền tệ. Đề bài giả định đã có `USER` với 1 `ACCOUNT` lưu số dư dạng `float` và các `TRANSACTION` (giao dịch thu/chi) ghi nhận vào tài khoản đó — nếu không có sẵn cột `amount_float` thì yêu cầu "migrate sang amount_cents" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ giao dịch và số dư, không suy diễn thêm ngân sách/báo cáo/nhắc nhở không liên quan.

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ TRANSACTION_ : "records into"

    USER {
        string user_id PK
        string full_name
        string email
    }

    ACCOUNT {
        string account_id PK
        string user_id FK
        float balance_float
    }

    TRANSACTION_ {
        string transaction_id PK
        string account_id FK
        float amount_float
        string type
        datetime created_at
    }
```

---

<a id="2-base-create-transaction-sqd"></a>

# Sequence Diagram — Base: Create Transaction

Đây là **base**, flow "tạo giao dịch thu/chi" — tiền đề bắt buộc cho việc đổi đơn vị lưu trữ: chính flow này ghi `amount_float` vào `TRANSACTION` và cập nhật `balance_float` của `ACCOUNT`, là nguồn dữ liệu phải được dual-write sang `amount_cents`/`balance_cents` ở enhance.

```mermaid
sequenceDiagram
    actor User
    participant App as Expense App Service
    participant DB as Account + Transaction Tables

    User->>App: Record transaction (amount, type)
    App->>DB: BEGIN TRANSACTION
    App->>DB: Lock account row for update
    App->>DB: INSERT INTO transactions (amount_float, type)
    App->>DB: UPDATE accounts SET balance_float = balance_float +/- amount_float
    App->>DB: COMMIT
    DB-->>App: New balance_float
    App-->>User: Transaction recorded, updated balance shown
```

---

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

---

# Sequence Diagram — Enhance: Create Transaction

Đây là **enhance**, flow "tạo giao dịch" đã tồn tại ở base ([2-base-create-transaction-sqd.md](#2-base-create-transaction-sqd)) nay thay đổi: mọi giao dịch mới phải ghi đồng thời `amount_float` và `amount_cents` (round-half-up) trong cùng 1 transaction, dùng cùng 1 lock trên dòng số dư, để race condition giữa giao dịch nạp tiền và rút tiền gần như đồng thời không làm lệch 2 cột.

```mermaid
sequenceDiagram
    actor UserA as User (nạp tiền, giao dịch A)
    actor UserB as User (rút tiền, giao dịch B)
    participant App as Expense App Service
    participant DB as Account + Transaction Tables

    par Race condition, 2 giao dịch gần như đồng thời trên cùng 1 account
        UserA->>App: Record income transaction A
    and
        UserB->>App: Record expense transaction B
    end

    App->>DB: Transaction A - BEGIN, lock account row for update
    App->>DB: Transaction A - INSERT transaction (amount_float, amount_cents = round_half_up(amount_float * 100))
    App->>DB: Transaction A - UPDATE balance_float and balance_cents together
    App->>DB: Transaction A - COMMIT, release lock

    App->>DB: Transaction B - BEGIN, lock account row for update (chờ lock A giải phóng)
    App->>DB: Transaction B - INSERT transaction (amount_float, amount_cents)
    App->>DB: Transaction B - UPDATE balance_float and balance_cents together
    App->>DB: Transaction B - COMMIT, release lock

    Note over DB: Cùng 1 lock đảm bảo không có trạng thái chỉ 1 cột được update
    DB-->>App: Final balance_float and balance_cents consistent
    App-->>UserA: Transaction A recorded
    App-->>UserB: Transaction B recorded
```

---

# Sequence Diagram — Enhance: Backfill Historical Transactions

Đây là **enhance**, flow hoàn toàn mới: job backfill tính lại `amount_cents` từ `amount_float` gốc theo batch ở giờ traffic thấp, validate tổng số dư sau mỗi batch và tự tạm dừng nếu tỷ lệ lệch vượt ngưỡng thay vì chạy tiếp và làm sai lệch số dư hàng loạt.

```mermaid
sequenceDiagram
    participant Job as Backfill Job
    participant Batch as Migration Batch Store
    participant DB as Account + Transaction Tables
    participant Alert as Alert Service

    Job->>Job: Check current traffic level, only proceed if low
    Job->>Batch: Read last_transaction_id_processed
    Batch-->>Job: Resume cursor

    loop Until no more transactions or job paused
        Job->>DB: SELECT next batch of transactions WHERE id > cursor
        DB-->>Job: Batch of transactions with amount_float

        loop for each transaction in batch
            Job->>Job: Compute amount_cents = round_half_up(amount_float * 100)
            Job->>DB: UPDATE transactions SET amount_cents
        end

        Job->>DB: Recompute balance_cents for affected accounts from amount_cents sum
        Job->>DB: Compare balance_float (as cents) vs balance_cents per user
        alt Lệch trong sai số cho phép, dưới 1 cent
            Job->>Batch: Update last_transaction_id_processed, continue
        else Lệch vượt ngưỡng cho phép
            Job->>Alert: Trigger alert, mismatch exceeds threshold
            Job->>Job: Pause backfill, do not proceed to next batch
        end
    end

    Job-->>Job: Backfill completed or paused pending investigation
```

---

# Sequence Diagram — Enhance: Rollback via Feature Flag

Đây là **enhance**, flow hoàn toàn mới: khi hệ thống đã chuyển đọc số dư sang `balance_cents` nhưng phát hiện lỗi nghiêm trọng, ops phải có cách quay lại đọc `balance_float` ngay lập tức bằng feature flag, không cần deploy lại code.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Flag as Read Source Flag Service
    participant App as Expense App Service
    actor User

    Note over App: Đang đọc số dư từ balance_cents (trạng thái sau cutover)

    Ops->>Flag: Phát hiện lỗi nghiêm trọng, set read_source = amount_float
    Flag-->>Ops: Flag updated, propagated ngay lập tức

    User->>App: Xem số dư tài khoản
    App->>Flag: Check current read_source flag
    Flag-->>App: read_source = amount_float
    App->>App: Đọc balance_float thay vì balance_cents
    App-->>User: Hiển thị số dư từ cột float, không cần deploy lại code

    Note over Ops,Flag: Sau khi khắc phục xong lỗi ở cột cents, ops set lại flag để chuyển đọc sang cents
```
