# Cập nhật điểm số và thứ hạng leaderboard trong game/app học tập

**Hệ thống:** Một app học tập gamification, nơi hoàn thành bài học cập nhật điểm user và một job nền tính lại thứ hạng leaderboard theo nhóm/lớp mỗi vài phút.

**Vai trò của flow:** Transaction cộng điểm khi user hoàn thành bài học và job tính lại rank (đọc/update nhiều user trong 1 nhóm) cạnh tranh lock trên bảng `user_score`, cần tránh deadlock giữa update đơn lẻ tần suất cao và job batch định kỳ.

**Yêu cầu cụ thể:**
- Yêu cầu job tính rank phải lock theo thứ tự user_id tăng dần trong từng nhóm, và xử lý theo từng nhóm nhỏ (không lock toàn bảng `user_score` một lần) để giảm collision với các transaction cộng điểm đang chạy real-time.
- Chỉ ra rủi ro cụ thể: nếu 2 user trong cùng nhóm cùng hoàn thành bài học đúng lúc job rank đang chạy, cả 2 transaction cộng điểm và transaction job đều có thể chờ nhau — yêu cầu retry tự động cho transaction cộng điểm (ưu tiên UX real-time của user) khi gặp deadlock, tối đa 2 lần trong 500ms.
- Chọn isolation level `READ COMMITTED` cho transaction cộng điểm để có UX nhanh, nhưng job tính rank cần đảm bảo tính nhất quán tương đối (không bắt buộc real-time chính xác tuyệt đối) — giải thích trade-off này bằng ví dụ cụ thể.
- Thiết kế cơ chế "lock timeout" ngắn (ví dụ 1 giây) cho transaction cộng điểm, để nếu job rank đang giữ lock lâu, user vẫn nhận được phản hồi (có thể là "điểm sẽ cập nhật sau ít giây") thay vì trải nghiệm bị treo.
- Đo và log tỷ lệ deadlock/lock-timeout theo giờ cao điểm (khi nhiều user học cùng lúc) để xác định xem có cần điều chỉnh tần suất chạy job rank hay không.

---

# Base ERD — Điểm số và leaderboard chưa có cơ chế chống deadlock

Đây là **base**: trạng thái hệ thống app học tập gamification *trước khi* áp dụng thứ tự lock theo batch nhỏ, retry và lock timeout. Suy luận từ đề bài, base đã có `USER_GROUP` (nhóm/lớp), `USER` thuộc 1 nhóm, `USER_SCORE` lưu điểm và rank hiện tại của từng user, và `LESSON_COMPLETION` ghi nhận mỗi lần hoàn thành bài học (nguồn cộng điểm). Base chưa phân biệt cách job tính rank và transaction cộng điểm truy cập `USER_SCORE`, cả 2 đều thao tác trực tiếp không có chiến lược tránh tranh chấp.

```mermaid
erDiagram
    USER_GROUP ||--o{ USER : contains
    USER ||--|| USER_SCORE : has
    USER ||--o{ LESSON_COMPLETION : completes

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
        datetime rank_updated_at
    }
    LESSON_COMPLETION {
        string id PK
        string user_id FK
        string lesson_id
        int points_earned
        datetime completed_at
    }
```

---

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

---

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

---

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

---

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

---

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

---

# Enhance sequence — Xem leaderboard (chấp nhận rank trễ vài phút, đổi lấy UX nhanh)

Đây là **enhance**, flow mới minh họa trade-off giữa isolation level nhanh cho transaction cộng điểm và độ trễ chấp nhận được của rank. Đáp ứng yêu cầu 3 của đề bài: `READ COMMITTED` cho transaction cộng điểm giúp UX nhanh, nhưng rank hiển thị cho user có thể trễ tới khi job batch tiếp theo chạy xong, không cần chính xác tuyệt đối theo thời gian thực.

```mermaid
sequenceDiagram
    actor User
    participant App as Learning App
    participant DB as Database
    participant Score as USER_SCORE (user X)

    User->>App: Hoàn thành bài học lúc 10:00:05
    App->>DB: Transaction cộng điểm (READ COMMITTED), COMMIT ngay
    DB->>Score: total_score = 120 (cộng thêm điểm mới)
    Note over Score: rank vẫn giữ giá trị cũ, chưa được tính lại

    User->>App: Mở màn hình leaderboard lúc 10:00:10
    App->>DB: Đọc USER_SCORE (user X)
    DB-->>App: total_score=120, rank=5 (rank_updated_at=09:57, trước khi job batch tiếp theo chạy)
    App-->>User: Hiển thị điểm mới nhất (120) nhưng rank vẫn là 5 (có thể chưa phản ánh đúng vị trí thực tế)

    Note over App,DB: Ví dụ cụ thể: user X vừa vượt qua user Y về điểm số, nhưng rank hiển thị vẫn xếp X sau Y cho tới khi Rank Job batch kế tiếp chạy (vài phút sau) và cập nhật lại rank_updated_at, đây là đánh đổi chấp nhận được vì rank không phải dữ liệu cần chính xác tức thời như điểm số
```

---

# Enhance sequence — Đo và log tỷ lệ deadlock/lock-timeout theo giờ cao điểm

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Đáp ứng yêu cầu 5 của đề bài: đo và log tỷ lệ deadlock/lock-timeout theo giờ để xác định có cần điều chỉnh tần suất chạy job rank hay không.

```mermaid
sequenceDiagram
    participant App as Learning App
    participant Job as Rank Recalculation Job
    participant Stats as DEADLOCK_STATS
    participant Ops as Đội vận hành

    loop Mỗi lần transaction cộng điểm gặp deadlock hoặc lock timeout
        App->>Stats: Tăng deadlock_count/lock_timeout_count cho hour_window hiện tại, context=score_update
    end

    loop Mỗi lần batch của Rank Job gặp deadlock hoặc lock timeout
        Job->>Stats: Tăng deadlock_count/lock_timeout_count cho hour_window hiện tại, context=rank_job_batch
    end

    Ops->>Stats: Truy vấn tỷ lệ deadlock/lock-timeout theo từng giờ trong ngày
    Stats-->>Ops: Trả về báo cáo, ví dụ giờ cao điểm 19h-21h có tỷ lệ deadlock cao gấp 5 lần giờ thường

    alt Tỷ lệ deadlock/lock-timeout ở giờ cao điểm vượt ngưỡng chấp nhận được
        Ops->>Job: Điều chỉnh giảm tần suất chạy job rank trong khung giờ cao điểm, hoặc giảm kích thước batch
    else Tỷ lệ vẫn trong ngưỡng bình thường
        Ops->>Job: Giữ nguyên cấu hình hiện tại
    end
```
