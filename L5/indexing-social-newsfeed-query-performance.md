# Mạng xã hội tối ưu truy vấn newsfeed theo follow và thời gian

**Hệ thống:** Mạng xã hội hiển thị newsfeed gồm bài viết từ những người user đang follow, sắp xếp theo thời gian đăng và/hoặc mức độ liên quan.

**Vai trò của flow:** Chọn đúng chiến lược index (và ranh giới khi nào chuyển từ pull-query sang fan-out/push) để trả feed nhanh dù số lượng người follow và tổng số bài viết tăng lớn.

**Yêu cầu cụ thể:**
- Cách tiếp cận pull (JOIN bảng follows với posts, ORDER BY created_at) cần composite index (followee_id, created_at) trên bảng posts, nhưng với user follow rất nhiều tài khoản hoặc follow một celebrity có lượng bài viết khổng lồ, JOIN + index vẫn không đủ nhanh — cần xác định ngưỡng khi nào nên chuyển sang mô hình fan-out/push (ghi sẵn feed) thay vì tiếp tục tối ưu index cho pull-query.
- Bài viết bị xoá hoặc ẩn phải biến mất khỏi feed ngay lập tức, nhưng nếu chỉ dựa vào cột `is_deleted`/`is_hidden` không nằm trong index thì mỗi lần lọc vẫn phải quét thêm — cần đưa các cột trạng thái này vào composite/partial index để loại bỏ ngay ở tầng index thay vì lọc sau khi đã lấy dữ liệu.
- Khi feed cần xếp hạng theo độ liên quan (kết hợp engagement score, không chỉ theo thời gian đăng), điểm số này thay đổi liên tục theo lượt like/comment mới — B-tree index không tối ưu để sort theo một giá trị luôn biến động, cần cân nhắc pre-computed/materialized ranking được cập nhật định kỳ thay vì tính và sort real-time trên index thời gian.
- Ở các sự kiện viral, lượng bài viết mới insert tăng đột biến khiến index trên bảng posts (đặc biệt phần cuối theo created_at) bị ghi liên tục dẫn tới page split và phân mảnh — cần tính tới việc tuning fillfactor hoặc lịch bảo trì/rebuild index định kỳ để giữ hiệu năng ổn định.
- Danh sách follow/unfollow của user thay đổi liên tục, và bảng follows được truy vấn theo cả hai chiều (tìm những người tôi follow, và tìm những ai đang follow tôi) — cần quyết định thứ tự cột index (follower_id, followee_id) hay ngược lại dựa trên chiều truy vấn nào phổ biến hơn trong hệ thống, hoặc chấp nhận tạo cả hai index đánh đổi lấy overhead ghi.

---

# ERD - Base (trước khi tối ưu index cho newsfeed)

Đây là trạng thái **base**: mô hình dữ liệu tối thiểu để một mạng xã hội có newsfeed theo follow tồn tại, trước khi áp bất kỳ tối ưu index nào. Chỉ gồm `users`, `follows` (quan hệ hai chiều) và `posts` với các cột trạng thái `is_deleted`/`is_hidden` cơ bản (đã có nhưng chưa được đưa vào index) - đây chính là phần dữ liệu mà đề bài (enhance) sẽ tác động trực tiếp.

```mermaid
erDiagram
    USERS ||--o{ FOLLOWS : follows_as_follower
    USERS ||--o{ FOLLOWS : followed_as_followee
    USERS ||--o{ POSTS : authors

    USERS {
        bigint id PK
        varchar username
        varchar display_name
        timestamp created_at
    }

    FOLLOWS {
        bigint follower_id FK "user thuc hien follow"
        bigint followee_id FK "user duoc follow"
        timestamp created_at
    }

    POSTS {
        bigint id PK
        bigint author_id FK
        text content
        boolean is_deleted "co san nhung chua nam trong index nao"
        boolean is_hidden "co san nhung chua nam trong index nao"
        timestamp created_at "chua co composite index (author_id, created_at)"
    }
```

---

# Sequence Diagram - Base: Xem newsfeed

Đây là trạng thái **base**, flow "xem newsfeed" bằng pull-query đơn giản: JOIN `follows` với `posts`, lọc `is_deleted`/`is_hidden` và sắp xếp theo `created_at` mà chưa có composite/partial index hỗ trợ. Flow này được chọn vì đây chính là truy vấn mà toàn bộ đề bài (enhance) nhắm tới tối ưu.

```mermaid
sequenceDiagram
    actor U as User
    participant API as Feed API
    participant DB as Postgres (posts, follows)

    U->>API: GET /feed
    API->>DB: SELECT p.* FROM posts p JOIN follows f ON p.author_id = f.followee_id WHERE f.follower_id = :me AND p.is_deleted = false AND p.is_hidden = false ORDER BY p.created_at DESC LIMIT 20
    Note over DB: Chua co composite index (followee_id, created_at) tren posts
    Note over DB: Postgres phai quet nhieu row roi moi loc is_deleted/is_hidden va sort
    DB-->>API: danh sach post da loc sau khi quet
    API-->>U: tra ve feed theo thoi gian
```

---

# Sequence Diagram - Base: Follow / Unfollow

Đây là trạng thái **base**, flow "follow/unfollow" và truy vấn bảng `follows` theo cả hai chiều (tôi follow ai, ai follow tôi) khi chưa quyết định thứ tự cột index. Flow này được chọn vì đề bài đặt thẳng câu hỏi nên đánh index `(follower_id, followee_id)` hay ngược lại, và base cần cho thấy vấn đề khi chỉ có (hoặc chưa có) index tối ưu cho một chiều.

```mermaid
sequenceDiagram
    actor U as User
    participant API as Social API
    participant DB as Postgres (follows)

    U->>API: POST /follow/:targetId
    API->>DB: INSERT INTO follows(follower_id, followee_id) VALUES (:me, :targetId)
    DB-->>API: OK
    API-->>U: 200 da follow

    U->>API: GET /following (nhung nguoi toi dang follow)
    API->>DB: SELECT followee_id FROM follows WHERE follower_id = :me
    DB-->>API: danh sach following
    API-->>U: tra ve danh sach following

    U->>API: GET /followers (ai dang follow toi)
    API->>DB: SELECT follower_id FROM follows WHERE followee_id = :me
    Note over DB: Chua ro index nao toi uu cho chieu truy van theo followee_id
    Note over DB: Co the phai quet toan bang neu bang follows lon
    DB-->>API: danh sach followers (co the cham)
    API-->>U: tra ve danh sach followers

    U->>API: DELETE /follow/:targetId
    API->>DB: DELETE FROM follows WHERE follower_id = :me AND followee_id = :targetId
    DB-->>API: OK
    API-->>U: 200 da unfollow
```

---

# ERD - Enhance (sau khi tối ưu index cho newsfeed)

Đây là trạng thái **enhance**: giữ nguyên `users`, `follows`, `posts` từ base, cộng thêm những gì đề bài yêu cầu - composite/partial index trên `posts` gồm cả `is_deleted`/`is_hidden`, index cho cả hai chiều của `follows`, cột `engagement_score` pre-computed thay vì sort real-time, và bảng `feed_inbox` phục vụ mô hình fan-out/push khi pull-query không còn đủ nhanh (follow quá nhiều người hoặc theo dõi celebrity). `likes`/`comments` chỉ được thêm ở mức tối thiểu vì là nguồn dữ liệu đầu vào để tính `engagement_score`.

```mermaid
erDiagram
    USERS ||--o{ FOLLOWS : follows_as_follower
    USERS ||--o{ FOLLOWS : followed_as_followee
    USERS ||--o{ POSTS : authors
    USERS ||--o{ FEED_INBOX : owns_feed
    POSTS ||--o{ FEED_INBOX : fanned_out_to
    POSTS ||--o{ LIKES : receives
    POSTS ||--o{ COMMENTS : receives
    USERS ||--o{ LIKES : makes
    USERS ||--o{ COMMENTS : makes

    USERS {
        bigint id PK
        varchar username
        varchar display_name
        timestamp created_at
    }

    FOLLOWS {
        bigint follower_id FK "trong idx (follower_id, followee_id) cho chieu toi follow ai"
        bigint followee_id FK "trong idx (followee_id, follower_id) cho chieu ai follow toi"
        timestamp created_at
    }

    POSTS {
        bigint id PK
        bigint author_id FK "trong composite/partial idx (author_id, created_at) WHERE is_deleted=false AND is_hidden=false"
        text content
        boolean is_deleted "nam trong partial index de loai ngay o tang index"
        boolean is_hidden "nam trong partial index de loai ngay o tang index"
        numeric engagement_score "pre-computed, cap nhat dinh ky boi batch job, dung de ORDER BY thay vi B-tree tren gia tri bien dong"
        timestamp created_at "phan cuoi index duoc tuning fillfactor/rebuild dinh ky khi insert dot bien luc viral"
    }

    FEED_INBOX {
        bigint id PK
        bigint owner_id FK "user se doc feed nay - ket qua fan-out/push"
        bigint post_id FK
        timestamp created_at
    }

    LIKES {
        bigint id PK
        bigint post_id FK
        bigint user_id FK
        timestamp created_at
    }

    COMMENTS {
        bigint id PK
        bigint post_id FK
        bigint user_id FK
        timestamp created_at
    }
```

So với base: `posts` thêm cột `engagement_score` và được đánh composite/partial index gồm cả `is_deleted`/`is_hidden`; `follows` được đánh index cho cả hai chiều; thêm mới `feed_inbox` (mô hình push cho trường hợp vượt ngưỡng pull-query) và `likes`/`comments` (nguồn tính engagement score).

---

# Sequence Diagram - Enhance: Xem newsfeed

Đây là trạng thái **enhance** của flow "xem newsfeed" đã có ở base. So với base, flow này thay đổi ở: (1) dùng composite/partial index `(author_id, created_at) WHERE is_deleted=false AND is_hidden=false` nên bài xoá/ẩn bị loại ngay ở tầng index thay vì lọc sau khi quét, (2) có nhánh kiểm tra ngưỡng để quyết định tiếp tục pull-query hay đọc từ `feed_inbox` đã fan-out sẵn khi user follow quá nhiều người hoặc follow celebrity, (3) có thể sắp theo `engagement_score` pre-computed thay vì tính real-time.

```mermaid
sequenceDiagram
    actor U as User
    participant API as Feed API
    participant DB as Postgres (posts, follows, feed_inbox)

    U->>API: GET /feed
    API->>DB: kiem tra so nguoi dang follow va co follow celebrity khong
    alt duoi nguong, pull-model van du nhanh
        API->>DB: SELECT p.* FROM posts p JOIN follows f ON p.author_id = f.followee_id WHERE f.follower_id = :me ORDER BY p.created_at DESC LIMIT 20
        Note over DB: Dung composite/partial index (author_id, created_at) WHERE is_deleted=false AND is_hidden=false
        Note over DB: Bai xoa/an bi loai ngay o tang index, khong can loc lai sau khi lay du lieu
        DB-->>API: danh sach post da loc san
    else vuot nguong, follow rat nhieu nguoi hoac follow celebrity
        API->>DB: SELECT * FROM feed_inbox WHERE owner_id = :me ORDER BY created_at DESC LIMIT 20
        Note over DB: Feed da duoc ghi san tu truoc (fan-out/push), khong can JOIN + sort luc doc
        DB-->>API: danh sach post da fan-out san
    end
    opt can xep hang theo do lien quan
        API->>DB: sap xep lai theo engagement_score da pre-computed thay vi tinh real-time
    end
    API-->>U: tra ve feed
```

---

# Sequence Diagram - Enhance: Follow / Unfollow

Đây là trạng thái **enhance** của flow "follow/unfollow" đã có ở base. So với base, flow này thay đổi ở: bảng `follows` nay duy trì index cho cả hai chiều `(follower_id, followee_id)` và `(followee_id, follower_id)` đánh đổi lấy chi phí ghi, đồng thời nếu người được follow đang ở diện fan-out/push thì follow/unfollow còn phải backfill hoặc dọn dẹp `feed_inbox` tương ứng.

```mermaid
sequenceDiagram
    actor U as User
    participant API as Social API
    participant DB as Postgres (follows)
    participant FanOut as Fan-out Worker

    U->>API: POST /follow/:targetId
    API->>DB: INSERT INTO follows(follower_id, followee_id)
    Note over DB: Ghi dong thoi duy tri 2 index (follower_id, followee_id) va (followee_id, follower_id)
    Note over DB: Toi uu ca hai chieu truy van, danh doi them chi phi ghi
    DB-->>API: OK
    alt targetId dang o dien fan-out/push
        API->>FanOut: backfill cac bai viet gan day cua targetId vao feed_inbox cua :me
        FanOut-->>API: da backfill xong
    end
    API-->>U: 200 da follow

    U->>API: DELETE /follow/:targetId
    API->>DB: DELETE FROM follows WHERE follower_id = :me AND followee_id = :targetId
    DB-->>API: OK
    alt targetId dang duoc fan-out cho :me
        API->>FanOut: xoa cac ban ghi cua targetId khoi feed_inbox cua :me
        FanOut-->>API: da don dep xong
    end
    API-->>U: 200 da unfollow

    U->>API: GET /followers (ai dang follow toi)
    API->>DB: SELECT follower_id FROM follows WHERE followee_id = :me
    Note over DB: Dung index (followee_id, follower_id) rieng cho chieu nay
    Note over DB: Nhanh hon dang ke so voi base khi chua co index toi uu cho chieu nay
    DB-->>API: danh sach followers
    API-->>U: tra ve danh sach
```

---

# Sequence Diagram - Enhance: Đăng bài và fan-out ghi feed

Đây là flow hoàn toàn mới phát sinh từ enhance, chưa tồn tại ở base. Flow này thể hiện ngưỡng chuyển từ pull-query sang mô hình fan-out/push mà đề bài đặt ra: khi số follower dưới ngưỡng, hệ thống ghi sẵn bài viết vào `feed_inbox` của từng follower ngay lúc đăng bài (đánh đổi write amplification để đọc feed nhanh); khi tác giả là celebrity có quá nhiều follower, bỏ qua fan-out ghi trước và để pull-query xử lý lúc đọc (mô hình hybrid).

```mermaid
sequenceDiagram
    actor U as Author
    participant API as Post API
    participant DB as Postgres (posts)
    participant FanOut as Fan-out Worker
    participant Inbox as Postgres (feed_inbox)

    U->>API: POST /posts (dang bai viet moi)
    API->>DB: INSERT INTO posts(author_id, content, created_at)
    DB-->>API: post_id
    API-->>U: 201 da dang bai

    API->>FanOut: enqueue fan-out job cho post_id
    FanOut->>DB: dem so luong follower cua author_id
    DB-->>FanOut: so luong follower

    alt so follower duoi nguong fan-out
        FanOut->>Inbox: INSERT feed_inbox(owner_id, post_id) cho tung follower
        Inbox-->>FanOut: da ghi xong
    else tac gia la celebrity, vuot xa nguong
        Note over FanOut: Bo qua fan-out ghi truoc de tranh write amplification qua lon
        Note over FanOut: Follower cua celebrity se doc bai nay qua pull-query luc xem feed
    end
```

---

# Sequence Diagram - Enhance: Xoá / ẩn bài viết

Đây là flow hoàn toàn mới phát sinh từ enhance, chưa được vẽ riêng ở base. Flow này thể hiện trực tiếp yêu cầu "bài viết bị xoá hoặc ẩn phải biến mất khỏi feed ngay lập tức" nhờ `is_deleted`/`is_hidden` đã nằm trong composite/partial index, nên không còn phải lọc lại sau khi quét dữ liệu như ở base.

```mermaid
sequenceDiagram
    actor U as Author
    participant API as Post API
    participant DB as Postgres (posts)
    participant Inbox as Postgres (feed_inbox)
    actor V as Viewer khac

    U->>API: PATCH /posts/:id (is_deleted=true hoac is_hidden=true)
    API->>DB: UPDATE posts SET is_deleted = true WHERE id = :id
    Note over DB: Composite/partial index WHERE is_deleted=false AND is_hidden=false tu dong loai row nay khoi index
    DB-->>API: OK
    opt bai da tung duoc fan-out truoc do
        API->>Inbox: (async) loc bo hoac join kem dieu kien is_deleted/is_hidden khi doc feed_inbox
        Inbox-->>API: da xu ly
    end
    API-->>U: 200 da xoa/an bai viet

    V->>API: GET /feed (ngay sau do)
    API->>DB: SELECT ... WHERE f.follower_id = :viewer ORDER BY p.created_at DESC LIMIT 20
    Note over DB: Index scan bo qua ngay bai vua bi danh dau xoa/an, khong can loc them o tang ung dung
    DB-->>API: feed khong con bai vua xoa/an
    API-->>V: tra ve feed moi nhat
```

---

# Sequence Diagram - Enhance: Batch job tính engagement score

Đây là flow hoàn toàn mới phát sinh từ enhance, chưa tồn tại ở base. Flow này giải quyết vấn đề đề bài nêu: khi feed cần xếp hạng theo mức độ liên quan, điểm số biến động liên tục theo like/comment mới khiến B-tree index không tối ưu để sort real-time, nên cần một batch job định kỳ pre-computed `engagement_score` thay vì tính lúc đọc feed.

```mermaid
sequenceDiagram
    participant Scheduler as Cron Scheduler
    participant Job as Engagement Score Job
    participant DB as Postgres (posts, likes, comments)

    Scheduler->>Job: kich hoat dinh ky (vi du moi 5-15 phut)
    Job->>DB: dem so luot like/comment moi cho tung bai viet gan day
    DB-->>Job: so lieu like/comment
    Job->>Job: tinh engagement_score (vi du trong so theo thoi gian dang, so luot like, comment)
    Job->>DB: UPDATE posts SET engagement_score = :score WHERE id = :post_id
    Note over DB: engagement_score duoc luu san (materialized), khong tinh real-time luc doc feed
    DB-->>Job: da cap nhat xong
    Note over Job: Flow xem newsfeed sau nay chi can ORDER BY engagement_score co san, khong phai JOIN/tinh toan like-comment ngay luc user xem feed
```

---

# Sequence Diagram - Enhance: Bảo trì index khi sự kiện viral

Đây là flow hoàn toàn mới phát sinh từ enhance, chưa tồn tại ở base. Flow này giải quyết vấn đề đề bài nêu: ở các sự kiện viral, lượng insert đột biến vào `posts` khiến phần cuối index theo `created_at` bị ghi liên tục dẫn tới page split và phân mảnh, nên cần tuning `fillfactor` và/hoặc lịch bảo trì/rebuild index định kỳ.

```mermaid
sequenceDiagram
    participant Monitor as Ops Monitor
    participant DB as Postgres (posts + index)
    participant Job as Index Maintenance Job

    Note over DB: Su kien viral, insert dot bien vao posts
    Note over DB: Phan cuoi index composite (author_id, created_at) bi ghi lien tuc
    DB->>Monitor: canh bao bloat/phan manh tang cao tren index
    Monitor->>Job: kich hoat quy trinh bao tri
    Job->>DB: kiem tra fillfactor hien tai cua bang posts
    alt fillfactor chua toi uu cho khoi luong ghi lon
        Job->>DB: ALTER TABLE posts SET (fillfactor = 70)
        Note over DB: Chua san khoang trong tren moi trang de giam page split khi insert/update lien tuc
    end
    Job->>DB: REINDEX CONCURRENTLY hoac chay theo lich rebuild index dinh ky
    DB-->>Job: index da duoc don dep, giam phan manh
    Job-->>Monitor: bao cao hoan tat bao tri
```
