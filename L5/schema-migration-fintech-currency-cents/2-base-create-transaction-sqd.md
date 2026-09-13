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
