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
