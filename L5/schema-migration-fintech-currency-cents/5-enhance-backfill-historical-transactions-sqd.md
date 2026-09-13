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
