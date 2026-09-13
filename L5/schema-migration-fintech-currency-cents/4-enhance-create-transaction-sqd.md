# Sequence Diagram — Enhance: Create Transaction

Đây là **enhance**, flow "tạo giao dịch" đã tồn tại ở base ([2-base-create-transaction-sqd.md](2-base-create-transaction-sqd.md)) nay thay đổi: mọi giao dịch mới phải ghi đồng thời `amount_float` và `amount_cents` (round-half-up) trong cùng 1 transaction, dùng cùng 1 lock trên dòng số dư, để race condition giữa giao dịch nạp tiền và rút tiền gần như đồng thời không làm lệch 2 cột.

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
