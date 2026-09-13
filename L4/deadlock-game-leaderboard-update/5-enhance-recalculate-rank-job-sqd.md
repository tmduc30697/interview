# Enhance sequence — Job tính lại rank (batch nhỏ, lock theo user_id tăng dần)

Đây là **enhance** của flow `recalculate-rank-job` đã có ở base. So với base (lock cả nhóm cùng lúc, thứ tự không kiểm soát), nay job chia nhóm thành nhiều `RANK_JOB_BATCH` nhỏ, mỗi batch chỉ lock 1 khoảng `user_id` liên tiếp theo thứ tự tăng dần, giảm đáng kể collision với các transaction cộng điểm real-time. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    participant Job as Rank Recalculation Job
    participant DB as Database
    participant B1 as RANK_JOB_BATCH (user_id 1-50)
    participant B2 as RANK_JOB_BATCH (user_id 51-100)
    actor UserA as User A (user_id=30, đang hoàn thành bài học)

    Job->>DB: Chia nhóm thành các batch theo user_id tăng dần (vd mỗi batch 50 user)
    Job->>DB: BEGIN Transaction cho batch 1 (user_id 1-50)
    DB->>B1: SELECT ... FOR UPDATE user_score theo user_id tăng dần trong batch 1
    B1-->>DB: Lock granted lần lượt theo thứ tự cố định

    par Đồng thời
        UserA->>DB: BEGIN Transaction cộng điểm (user_id=30, lock_timeout=1000ms)
        DB->>B1: SELECT ... FOR UPDATE user_score (user_id=30) - chờ lock, nhưng chỉ trong batch nhỏ đang chạy
    end

    DB->>DB: Job tính rank xong cho batch 1, COMMIT, giải phóng lock batch 1
    DB->>B1: User A nhận lock (user_id=30), cộng điểm, COMMIT

    Job->>DB: BEGIN Transaction cho batch 2 (user_id 51-100)
    DB->>B2: SELECT ... FOR UPDATE user_score theo user_id tăng dần trong batch 2

    Note over Job,B2: Vì mỗi batch chỉ lock 1 dải user_id nhỏ và luôn theo thứ tự tăng dần, thời gian giữ lock ngắn hơn nhiều so với lock cả nhóm, giảm mạnh khả năng deadlock với transaction cộng điểm
```
