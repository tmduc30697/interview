# Enhance sequence — Complete lesson (retry nhanh + lock timeout ngắn)

Đây là **enhance** của flow `complete-lesson` đã có ở base. So với base (không retry, không timeout, có thể chờ vô thời hạn nếu job rank đang giữ lock), nay transaction cộng điểm có `lock_timeout_ms=1000` và tự động retry tối đa 2 lần trong 500ms khi gặp deadlock, ưu tiên UX real-time của user. Đáp ứng yêu cầu 2 và 4 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Learning App
    participant DB as Database (isolation=READ COMMITTED, lock_timeout=1000ms)
    participant Score as USER_SCORE (user X, đang bị Rank Job batch giữ lock)

    User->>App: Hoàn thành bài học
    App->>DB: BEGIN Transaction cộng điểm (retry_attempt=1)
    DB->>Score: SELECT ... FOR UPDATE user_score (user X)
    Note over Score: Rank Job batch đang giữ lock user X lâu hơn 1 giây
    DB-->>App: Lock timeout sau 1000ms, hoặc deadlock được phát hiện, transaction rollback

    App->>App: retry_attempt=2, backoff nhỏ (trong tổng 500ms)
    App->>DB: BEGIN Transaction cộng điểm (retry_attempt=2)
    DB->>Score: SELECT ... FOR UPDATE user_score (user X)
    Score-->>DB: Rank Job batch đã COMMIT xong, lock được giải phóng
    DB->>Score: UPDATE total_score += points_earned
    DB-->>App: COMMIT thành công
    App-->>User: Hiển thị điểm mới

    alt Sau 2 lần retry trong 500ms vẫn thất bại
        App-->>User: "Điểm sẽ được cập nhật sau ít giây" (không để user chờ treo)
    end
```
