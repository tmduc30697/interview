# Base sequence — Job tính lại rank (lock cả nhóm cùng lúc, không sort user_id)

Đây là **base**, flow job nền tính lại thứ hạng leaderboard ở trạng thái hiện tại: job đọc/update toàn bộ user trong 1 nhóm cùng lúc, không sort theo `user_id`, không chia batch nhỏ. Khi chạy song song với các transaction cộng điểm real-time, đây chính là nguồn gốc rủi ro deadlock nêu ở yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    participant Job as Rank Recalculation Job
    participant DB as Database
    participant Grp as USER_SCORE (toàn bộ user trong nhóm, thứ tự không cố định)
    actor UserA as User A (đang hoàn thành bài học)
    actor UserB as User B (đang hoàn thành bài học)

    Job->>DB: BEGIN Transaction tính rank cho cả nhóm
    DB->>Grp: SELECT ... FOR UPDATE toàn bộ user_score trong nhóm (thứ tự duyệt tùy theo query plan)
    Grp-->>DB: Lock granted lần lượt theo thứ tự không kiểm soát

    par Đồng thời trong lúc job đang giữ lock
        UserA->>DB: BEGIN Transaction cộng điểm cho User A
        DB->>Grp: SELECT ... FOR UPDATE user_score (User A) - chờ lock
    and
        UserB->>DB: BEGIN Transaction cộng điểm cho User B
        DB->>Grp: SELECT ... FOR UPDATE user_score (User B) - chờ lock
    end

    Note over Job,Grp: Nếu job đã lock User B nhưng chưa lock User A, còn transaction cộng điểm của User A đang giữ lock và chờ lock User B (do thứ tự xử lý batch khác), có thể tạo vòng chờ chéo -> DEADLOCK
    DB-->>Job: Một trong 2 bên bị chọn làm nạn nhân, rollback, không retry
```
