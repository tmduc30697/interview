# Ngân hàng số đối soát giao dịch với đối tác thanh toán/mạng thẻ

**Hệ thống:** Một ngân hàng số (neobank) xử lý giao dịch thẻ và chuyển tiền qua các đối tác bên ngoài (mạng thẻ, cổng chuyển mạch liên ngân hàng).

**Vai trò của flow:** Đối soát cuối ngày giữa sổ cái (ledger) nội bộ của ngân hàng và file/báo cáo giao dịch nhận từ đối tác thanh toán để đảm bảo mọi giao dịch được ghi nhận đúng và số dư khách hàng chính xác.

**Yêu cầu cụ thể:**
- Đối soát phải xử lý được file batch từ đối tác thanh toán có thể đến muộn hoặc theo múi giờ khác với hệ thống nội bộ — phải định nghĩa rõ "ngày giao dịch" theo cùng một chuẩn giữa hai bên để tránh lệch ngày khi đối soát.
- Phát hiện chính xác các loại sai lệch khác nhau: giao dịch có ở đối tác nhưng thiếu trong ledger nội bộ, giao dịch có trong ledger nhưng đối tác không xác nhận, và giao dịch khớp nhưng sai số tiền — mỗi loại phải có quy trình xử lý và mức độ ưu tiên khác nhau vì đây là tiền thật của khách hàng.
- Giao dịch bị đối tác báo hoàn/đảo sau một khoảng thời gian (ví dụ tranh chấp thẻ) phải được xử lý đúng dù giao dịch gốc đã đối soát khớp từ trước, và số dư khách hàng phải được điều chỉnh đúng thời điểm phát hiện.
- Toàn bộ quy trình đối soát và mọi điều chỉnh số dư phát sinh từ đó phải có audit trail đầy đủ, không thể sửa trực tiếp số dư mà không qua bước ghi nhận giao dịch điều chỉnh tương ứng, để đáp ứng yêu cầu kiểm toán ngân hàng.
- Phải có ngưỡng cảnh báo tự động khi tổng sai lệch chưa xử lý trong ngày vượt quá một số tiền nhất định, yêu cầu can thiệp khẩn của đội vận hành trước khi đóng sổ ngày đó.

---

# ERD — Base (trước khi có đối soát cuối ngày)

Đây là **base**: mô hình dữ liệu suy luận cho hệ thống ngân hàng số *trước khi* có flow đối soát. Đề bài giả định ngân hàng đã xử lý giao dịch thẻ qua đối tác (mạng thẻ) và ghi sổ cái nội bộ — nếu không có sẵn Account/Transaction/Ledger thì "đối soát ledger nội bộ với báo cáo đối tác" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ giao dịch thẻ + ghi sổ, không suy diễn thêm các module không liên quan (chuyển khoản nội địa, tiết kiệm, thẻ tín dụng...).

```mermaid
erDiagram
    CUSTOMER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ CARD : has
    ACCOUNT ||--o{ TRANSACTION : "posts to"
    TRANSACTION ||--|| LEDGER_ENTRY : records

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
        string account_id FK
        decimal amount
        string entry_type
        decimal balance_after
        datetime created_at
    }
```

---

# Sequence Diagram — Base: Process Card Transaction

Đây là **base**, flow "xử lý giao dịch thẻ" — tiền đề bắt buộc cho đối soát: chính flow này tạo ra các `TRANSACTION` và `LEDGER_ENTRY` mà cuối ngày phải được đối chiếu với báo cáo của đối tác mạng thẻ. Đặc biệt, `partner_reference_id` được sinh ra ở bước này là khóa nối giữa hai phía khi đối soát.

```mermaid
sequenceDiagram
    actor Customer
    participant Merchant as Merchant/POS
    participant Network as Card Network (Partner)
    participant Bank as Bank Issuer Service
    participant Ledger as Ledger Service
    participant Account as Account Service

    Customer->>Merchant: Swipe/tap card for payment
    Merchant->>Network: Authorization request
    Network->>Bank: Forward authorization request (partner_reference_id)
    Bank->>Account: Check account balance and status
    Account-->>Bank: Balance sufficient, account active
    Bank->>Ledger: Create TRANSACTION + post LEDGER_ENTRY
    Ledger->>Account: Update balance (balance_after)
    Account-->>Ledger: Balance updated
    Ledger-->>Bank: Transaction recorded (transaction_id, partner_reference_id)
    Bank-->>Network: Authorization approved
    Network-->>Merchant: Approved
    Merchant-->>Customer: Payment successful

    Note over Bank,Network: Transaction is recorded internally right away
    Note over Bank,Network: Partner's own settlement report for this transaction arrives later (separate batch, end of day)
```

---

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

---

<a id="4-enhance-daily-reconciliation-sqd"></a>

# Sequence Diagram — Enhance: Daily Reconciliation

Đây là **enhance**, flow chính hoàn toàn mới mà đề bài yêu cầu: đối soát cuối ngày giữa ledger nội bộ và file batch từ đối tác mạng thẻ. Flow này thể hiện yêu cầu chuẩn hóa "ngày giao dịch" giữa hai múi giờ, phân loại 3 loại sai lệch với mức ưu tiên khác nhau, và ngưỡng cảnh báo tự động khi tổng sai lệch vượt mức — tất cả chưa tồn tại ở base.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Partner as Card Network (Partner)
    participant Recon as Reconciliation Service
    participant Ledger as Ledger Service
    participant Alert as Alert Service

    Partner->>Recon: Upload settlement batch file (business_date, partner_timezone)
    Note over Recon: File may arrive late, or after internal day-close cutoff

    Recon->>Recon: Normalize each record's transaction_datetime to internal standard business_date
    Recon->>Recon: Create RECONCILIATION_RUN for business_date

    loop for each PARTNER_SETTLEMENT_RECORD
        Recon->>Ledger: Look up TRANSACTION by partner_reference_id
        alt matched, amount equal
            Ledger-->>Recon: Transaction found, amount matches
            Recon->>Recon: Mark record as reconciled, no discrepancy
        else exists at partner, missing in ledger
            Ledger-->>Recon: No matching transaction found
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=missing_in_ledger, priority=high)
        else exists in ledger, partner did not confirm
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=missing_in_partner, priority=medium)
        else matched but amount differs
            Ledger-->>Recon: Transaction found, amount mismatch
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=amount_mismatch, priority=high)
        end
    end

    Recon->>Recon: Sum total_discrepancy_amount for the run
    alt total_discrepancy_amount exceeds threshold
        Recon->>Alert: Trigger RECONCILIATION_ALERT (threshold, actual amount)
        Alert-->>Ops: Urgent notification, day-close blocked
        Ops->>Recon: Acknowledge alert, begin triage
    else within threshold
        Recon->>Recon: Mark RECONCILIATION_RUN as completed
        Recon-->>Ops: Reconciliation summary, day-close allowed
    end
```

---

<a id="5-enhance-handle-discrepancy-sqd"></a>

# Sequence Diagram — Enhance: Handle Discrepancy

Đây là **enhance**, flow mới xử lý một `RECONCILIATION_DISCREPANCY` được tạo ra từ flow đối soát ([4-enhance-daily-reconciliation-sqd.md](#4-enhance-daily-reconciliation-sqd)). Flow này thể hiện yêu cầu "không thể sửa trực tiếp số dư mà không qua bước ghi nhận giao dịch điều chỉnh tương ứng" và audit trail đầy đủ — số dư khách hàng (vốn chỉ được cập nhật qua LEDGER_ENTRY ở base) nay chỉ có thể thay đổi thêm qua một ADJUSTMENT_TRANSACTION được audit.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Ledger as Ledger Service
    participant Account as Account Service
    participant Audit as Audit Service

    Ops->>Recon: Review open discrepancies, ordered by priority
    Recon-->>Ops: Discrepancy detail (type, amount_diff, related transaction/record)
    Ops->>Recon: Approve adjustment with reason

    Recon->>Recon: Create ADJUSTMENT_TRANSACTION (discrepancy_id, amount, reason, created_by)
    Recon->>Ledger: Post new LEDGER_ENTRY for the adjustment
    Ledger->>Account: Update balance (balance_after)
    Account-->>Ledger: Balance updated
    Ledger-->>Recon: Ledger entry posted

    Recon->>Audit: Log adjustment (before_state, after_state, actor, timestamp)
    Audit-->>Recon: Audit entry recorded

    Recon->>Recon: Mark RECONCILIATION_DISCREPANCY as resolved
    Recon-->>Ops: Discrepancy resolved, balance corrected
```

---

# Sequence Diagram — Enhance: Handle Partner Reversal

Đây là **enhance**, flow mới xử lý trường hợp đối tác báo hoàn/đảo một giao dịch (ví dụ tranh chấp thẻ) sau một khoảng thời gian, kể cả khi giao dịch gốc đã đối soát khớp từ trước ở flow [4-enhance-daily-reconciliation-sqd.md](#4-enhance-daily-reconciliation-sqd). Flow này tái sử dụng cơ chế điều chỉnh có audit trail giống [5-enhance-handle-discrepancy-sqd.md](#5-enhance-handle-discrepancy-sqd), nhưng khởi phát từ thông báo của đối tác thay vì từ chạy đối soát, và số dư phải được điều chỉnh đúng tại thời điểm phát hiện chứ không lùi lại ngày giao dịch gốc.

```mermaid
sequenceDiagram
    participant Partner as Card Network (Partner)
    participant Recon as Reconciliation Service
    participant Ledger as Ledger Service
    participant Account as Account Service
    participant Audit as Audit Service
    actor Ops as Ops Team

    Partner->>Recon: Send reversal/chargeback notice (partner_transaction_id, reason, reversal_amount)
    Recon->>Ledger: Look up original TRANSACTION by partner_transaction_id
    Ledger-->>Recon: Original transaction found, was already reconciled and matched

    Recon->>Recon: Create PARTNER_REVERSAL_NOTICE (linked to original_transaction_id)
    Recon->>Recon: Create ADJUSTMENT_TRANSACTION (reversal_id, amount=reversal_amount, reason)

    Recon->>Ledger: Post new LEDGER_ENTRY for the reversal, dated at detection time
    Ledger->>Account: Update balance (balance_after) as of now, original entry stays untouched
    Account-->>Ledger: Balance updated

    Recon->>Audit: Log reversal adjustment (before_state, after_state, actor=system, timestamp)
    Audit-->>Recon: Audit entry recorded

    Recon->>Recon: Mark PARTNER_REVERSAL_NOTICE as processed
    Recon-->>Ops: Notify reversal processed, customer balance adjusted
```
