# Enhance ERD — Batch nhỏ theo user_id, retry, lock timeout và đo tỷ lệ deadlock

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có thêm 2 entity mới và bổ sung trường theo dõi cấu hình vào các entity hiện có, ứng trực tiếp với các yêu cầu:

- `RANK_JOB_BATCH` (mới) — job tính rank chia nhóm thành nhiều batch nhỏ, mỗi batch lock user_id tăng dần, đáp ứng yêu cầu 1.
- `LESSON_COMPLETION` thêm `retry_count`, `max_retry`, `lock_timeout_ms` — đáp ứng yêu cầu 2 (retry tối đa 2 lần trong 500ms) và yêu cầu 4 (lock timeout 1 giây cho transaction cộng điểm).
- `USER_SCORE.isolation_level`/`rank_updated_at` (giữ nguyên nhưng nay phản ánh rank có thể hơi trễ) — minh họa trade-off ở yêu cầu 3 (READ COMMITTED, chấp nhận nhất quán tương đối).
- `DEADLOCK_STATS` (mới) — đo và log tỷ lệ deadlock/lock-timeout theo giờ, đáp ứng yêu cầu 5.

```mermaid
erDiagram
    USER_GROUP ||--o{ USER : contains
    USER ||--|| USER_SCORE : has
    USER ||--o{ LESSON_COMPLETION : completes
    USER_GROUP ||--o{ RANK_JOB_BATCH : "chia thành nhiều batch"
    RANK_JOB_BATCH ||--o{ DEADLOCK_STATS : "có thể ghi nhận"
    LESSON_COMPLETION ||--o{ DEADLOCK_STATS : "có thể ghi nhận"

    USER_GROUP {
        string id PK
        string name
    }
    USER {
        string id PK
        string group_id FK
        string name
    }
    USER_SCORE {
        string user_id PK, FK
        string group_id FK
        int total_score
        int rank
        string isolation_level "READ_COMMITTED"
        datetime rank_updated_at "có thể trễ vài phút so với total_score mới nhất"
    }
    LESSON_COMPLETION {
        string id PK
        string user_id FK
        string lesson_id
        int points_earned
        int retry_count
        int max_retry "2"
        int lock_timeout_ms "1000"
        datetime completed_at
    }
    RANK_JOB_BATCH {
        string id PK
        string group_id FK
        string batch_start_user_id "theo user_id tăng dần"
        string batch_end_user_id
        datetime started_at
        datetime finished_at
    }
    DEADLOCK_STATS {
        string id PK
        string hour_window
        string context "score_update|rank_job_batch"
        int deadlock_count
        int lock_timeout_count
    }
```
