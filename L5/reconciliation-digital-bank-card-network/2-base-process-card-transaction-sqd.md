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
