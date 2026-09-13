# Base sequence — Complete lesson (cộng điểm, không retry, không timeout)

Đây là **base**, flow cộng điểm khi user hoàn thành bài học ở trạng thái hiện tại: transaction lock trực tiếp dòng `USER_SCORE` của user đó, không có retry khi gặp lỗi deadlock, không có lock timeout — nếu job tính rank đang giữ lock lâu, transaction này sẽ chờ vô thời hạn hoặc lỗi thẳng ra user. Flow này là tiền đề cho yêu cầu 2 và 4 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Learning App
    participant DB as Database
    participant Score as USER_SCORE (user X)

    User->>App: Hoàn thành bài học
    App->>DB: BEGIN Transaction cộng điểm
    DB->>Score: SELECT ... FOR UPDATE user_score (user X)
    Note over Score: Nếu dòng này đang bị job tính rank giữ lock, transaction phải chờ không giới hạn thời gian
    Score-->>DB: Lock granted (giả sử không có tranh chấp)
    DB->>Score: UPDATE total_score += points_earned
    DB-->>App: COMMIT thành công
    App-->>User: Hiển thị điểm mới ngay lập tức

    Note over App,DB: Nếu gặp lỗi deadlock với job tính rank, transaction rollback và lỗi bị trả thẳng cho user, không có cơ chế retry
```
