# Cache feed/trang cá nhân cho mạng xã hội

**Hệ thống:** Mạng xã hội cache feed trang chủ và trang cá nhân (profile) của user để giảm tải khi hiển thị bài viết, tránh phải chạy lại aggregation nặng mỗi lần load.

**Vai trò của flow:** Invalidate đúng cache khi user đăng bài mới, xóa bài, sửa bài, đổi thông tin profile — đặc biệt phải fan-out invalidate/refresh cho cache feed của tất cả follower đang xem nội dung của user đó.

**Yêu cầu cụ thể:**
- Khi user đăng bài mới, cache feed của chính họ (profile page) phải invalidate ngay, nhưng cache feed của hàng chục nghìn follower thì không thể invalidate đồng bộ tức thời — cần chiến lược phân biệt giữa fan-out-on-write (đẩy trước) cho user ít follower và fan-out-on-read (tính khi đọc) cho celebrity account có follower cực lớn, tránh nghẽn khi 1 bài đăng làm invalidate hàng triệu cache feed cùng lúc.
- Khi user xóa 1 bài viết, phải đảm bảo bài đó biến mất khỏi cache feed của follower trong thời gian hợp lý dù dùng fan-out-on-read hay pull — không được để bài đã xóa vẫn hiển thị dai dẳng do cache feed của follower ít khi active nên hiếm khi được refresh.
- Race condition: user vừa đăng bài mới vừa sửa bài cũ gần như đồng thời — invalidation event có thể tới cache theo thứ tự khác thứ tự thực tế (do khác hàng đợi/khác node xử lý), cần cơ chế đảm bảo thứ tự (version/timestamp) để cache feed không áp dụng nhầm invalidate cũ đè lên update mới.
- Khi user đổi avatar/tên hiển thị, các bài post cũ của họ đang nằm rải rác trong cache feed của follower khác vẫn hiển thị thông tin cũ (do cache feed thường lưu kèm snapshot info tác giả) — quyết định rõ có chấp nhận độ trễ hiển thị (denormalized data lệch tạm thời) hay bắt buộc phải invalidate toàn bộ cache liên quan, đánh đổi chi phí fan-out rất lớn.
- Đo lường: độ trễ trung bình và p99 từ lúc user đăng/xóa bài tới lúc feed của follower phản ánh đúng, và tỉ lệ follower vẫn thấy nội dung cũ quá X phút sau invalidation event, dùng để đánh giá hiệu quả chiến lược fan-out đang chọn.

---

# Base ERD — Cache feed trước khi có chiến lược fan-out phân biệt

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có chiến lược fan-out phân biệt theo quy mô follower. Đề bài nói cache feed lưu kèm snapshot tác giả (denormalized) và được fan-out cho follower — nên base cần đủ: user (kèm follower_count), quan hệ follow, bài viết, và feed cache entry theo từng follower (fan-out-on-write áp dụng đồng loạt, không phân biệt quy mô). Chưa có entity nào phục vụ version/ordering, chiến lược fan-out kép, hay đo staleness — những thứ đó là phần enhance.

```mermaid
erDiagram
    USER ||--o{ FOLLOW : "là follower"
    USER ||--o{ FOLLOW : "là followee"
    USER ||--o{ POST : authors
    POST ||--o{ FEED_CACHE_ENTRY : "fanned out as"
    USER ||--o{ FEED_CACHE_ENTRY : "nhận vào feed của"

    USER {
        string id PK
        string name
        string avatar
        int follower_count
    }
    FOLLOW {
        string follower_id FK
        string followee_id FK
    }
    POST {
        string id PK
        string author_id FK
        string content
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }
    FEED_CACHE_ENTRY {
        string id PK
        string follower_id FK
        string post_id FK
        string author_snapshot_name
        string author_snapshot_avatar
        datetime cached_at
    }
```

---

# Base sequence — Publish post (fan-out-on-write đồng loạt)

Đây là **base**, flow "Đăng bài mới" ở trạng thái hiện tại — fan-out-on-write cho toàn bộ follower, không phân biệt tài khoản có bao nhiêu follower. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 1 của đề bài chính là sửa đúng lỗ hổng "1 bài đăng làm invalidate hàng triệu cache feed cùng lúc" ở đây.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant Follow as FOLLOW store
    participant FeedCache as FEED_CACHE_ENTRY store

    Author->>App: Đăng bài mới
    App->>DB: Tạo POST(author_id, content, created_at)
    App->>Follow: Lấy toàn bộ follower của Author
    Follow-->>App: Danh sách follower (có thể tới hàng triệu nếu là celebrity)
    loop Với mỗi follower (không phân biệt quy mô)
        App->>FeedCache: Push FEED_CACHE_ENTRY(follower_id, post_id, author_snapshot hiện tại)
    end
    App-->>Author: "Đăng bài thành công" (chỉ sau khi fan-out xong hoặc timeout)
    Note over App,FeedCache: Với tài khoản có hàng triệu follower, vòng lặp fan-out này rất chậm/tốn tài nguyên, có thể làm nghẽn hệ thống ghi
```

---

# Base sequence — Delete post (invalidate không đáng tin cậy)

Đây là **base**, flow "Xóa bài viết" ở trạng thái hiện tại — chỉ đụng tới cache của follower đang hoạt động, không chủ động lan tới toàn bộ. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài chính là sửa đúng lỗ hổng "bài đã xóa vẫn hiển thị dai dẳng" ở đây.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant FeedCache as FEED_CACHE_ENTRY store
    actor ActiveFollower as Follower đang hoạt động
    actor InactiveFollower as Follower ít hoạt động

    Author->>App: Xóa bài viết
    App->>DB: Cập nhật POST.deleted_at
    App->>FeedCache: Cố gắng invalidate FEED_CACHE_ENTRY, nhưng chỉ chạm tới cache của follower đang active gần đây
    FeedCache-->>ActiveFollower: Cache được refresh, bài đã xóa biến mất
    Note over FeedCache,InactiveFollower: Follower ít hoạt động không được đụng tới — FEED_CACHE_ENTRY của họ vẫn còn bài đã xóa, không có cơ chế quét/lan truyền chủ động nào khác
    App-->>Author: "Xóa thành công"
```

---

# Enhance ERD — sau khi có chiến lược fan-out kép + ordering + đo lường

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `POST` và `FEED_CACHE_ENTRY` thêm `version`/`post_version` — đảm bảo thứ tự, tránh invalidate cũ đè lên update mới.
- `FEED_FANOUT_JOB` (mới) — ghi rõ mỗi bài dùng chiến lược `fanout_on_write` (follower ít) hay `fanout_on_read` (celebrity), thay vì đồng loạt như base.
- `FEED_INVALIDATION_EVENT` (mới) — mang theo version, dùng để lan truyền publish/edit/delete có thứ tự.
- `PROFILE_CHANGE_POLICY` (mới) — quyết định rõ chấp nhận staleness hay force invalidate khi đổi avatar/tên hiển thị.
- `FEED_STALENESS_METRIC` (mới) — đo độ trễ trung bình/p99 và tỉ lệ follower còn thấy nội dung cũ.

```mermaid
erDiagram
    USER ||--o{ FOLLOW : "là follower"
    USER ||--o{ FOLLOW : "là followee"
    USER ||--o{ POST : authors
    POST ||--o{ FEED_CACHE_ENTRY : "fanned out as"
    USER ||--o{ FEED_CACHE_ENTRY : "nhận vào feed của"
    POST ||--o{ FEED_FANOUT_JOB : "processed by"
    POST ||--o{ FEED_INVALIDATION_EVENT : emits
    POST ||--o{ FEED_STALENESS_METRIC : measures

    USER {
        string id PK
        string name
        string avatar
        int follower_count
    }
    FOLLOW {
        string follower_id FK
        string followee_id FK
    }
    POST {
        string id PK
        string author_id FK
        string content
        int version
        datetime created_at
        datetime updated_at
        datetime deleted_at
    }
    FEED_CACHE_ENTRY {
        string id PK
        string follower_id FK
        string post_id FK
        int post_version
        string author_snapshot_name
        string author_snapshot_avatar
        datetime cached_at
    }
    FEED_FANOUT_JOB {
        string id PK
        string post_id FK
        string strategy "fanout_on_write | fanout_on_read"
        int target_follower_count
        string status
        datetime started_at
        datetime completed_at
    }
    FEED_INVALIDATION_EVENT {
        string id PK
        string post_id FK
        string event_type "publish | edit | delete"
        int version
        datetime triggered_at
    }
    PROFILE_CHANGE_POLICY {
        string id PK
        string field "avatar | display_name"
        string strategy "accept_staleness | force_invalidate"
    }
    FEED_STALENESS_METRIC {
        string id PK
        string post_id FK
        string event_type
        string follower_id
        datetime triggered_at
        datetime reflected_at
        int staleness_ms
    }
```

---

# Enhance sequence — Publish post (fan-out-on-write vs fan-out-on-read)

Đây là **enhance**, cùng flow "Publish post" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 1 của đề bài: phân biệt chiến lược theo follower_count thay vì fan-out đồng loạt cho tất cả.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant Job as FEED_FANOUT_JOB store
    participant Follow as FOLLOW store
    participant FeedCache as FEED_CACHE_ENTRY store

    Author->>App: Đăng bài mới
    App->>DB: Tạo POST(author_id, content, version=1)
    App->>App: Kiểm tra USER.follower_count của Author

    alt follower_count dưới ngưỡng (tài khoản thường)
        App->>Job: Tạo FEED_FANOUT_JOB(strategy=fanout_on_write)
        Job->>Follow: Lấy danh sách follower (số lượng nhỏ)
        Follow-->>Job: Danh sách follower
        loop Với mỗi follower
            Job->>FeedCache: Push FEED_CACHE_ENTRY(follower_id, post_id, post_version=1, author_snapshot)
        end
        Job->>Job: status=completed
    else follower_count vượt ngưỡng (celebrity account)
        App->>Job: Tạo FEED_FANOUT_JOB(strategy=fanout_on_read)
        Note over Job,FeedCache: Không push entry cho hàng triệu follower — chỉ đánh dấu bài này thuộc diện fanout_on_read
        Job->>Job: status=indexed_for_read (feed của follower sẽ tự tính bài này khi họ load feed, xem thêm cơ chế đọc feed)
    end

    App->>App: Publish FEED_INVALIDATION_EVENT (event_type=publish, version=1) — dùng cho ordering ở các flow khác
    App-->>Author: "Đăng bài thành công"
```

---

# Enhance sequence — Delete post (tombstone chủ động + đo staleness)

Đây là **enhance**, cùng flow "Delete post" đã có ở base nhưng nay thay đổi theo yêu cầu 2 và 5 của đề bài: chủ động quét và xoá entry ở toàn bộ follower thuộc diện fanout_on_write (không chỉ follower đang hoạt động), và đo lại staleness thực tế cho từng follower.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant FeedCache as FEED_CACHE_ENTRY store
    participant Metric as FEED_STALENESS_METRIC store

    Author->>App: Xóa bài viết
    App->>DB: Cập nhật POST.deleted_at, version += 1
    App->>App: Publish FEED_INVALIDATION_EVENT (event_type=delete, version mới)

    alt Bài thuộc diện fanout_on_write
        App->>FeedCache: Quét toàn bộ FEED_CACHE_ENTRY(post_id) đang tồn tại, không chỉ của follower đang active
        loop Với mỗi entry tìm thấy
            FeedCache->>FeedCache: Kiểm tra version event > post_version đang lưu trước khi xoá (tránh đè nhầm nếu có update mới hơn)
            FeedCache->>FeedCache: Xoá/tombstone entry
            FeedCache->>Metric: Ghi FEED_STALENESS_METRIC (triggered_at=lúc xoá, reflected_at=lúc entry bị xoá, staleness_ms)
        end
        Note over FeedCache: Follower ít hoạt động cũng được dọn ngay, không phải chờ họ tự load lại mới refresh như base
    else Bài thuộc diện fanout_on_read
        Note over FeedCache: Không có entry vật lý nào cần xoá — lần đọc feed tiếp theo của follower tự loại bỏ bài đã xóa do kiểm tra deleted_at tại thời điểm đọc
    end

    App-->>Author: "Xóa thành công"
```

---

# Enhance sequence — Concurrent publish/edit ordering

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có cơ chế ordering nào). Đáp ứng yêu cầu thứ 3 của đề bài: 2 invalidation event cho cùng 1 bài tới cache theo thứ tự khác thứ tự thực tế (do khác hàng đợi/khác node xử lý), phải dùng version để tránh event cũ đè nhầm lên update mới.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant Queue as Invalidation Queue (nhiều worker/node)
    participant FeedCache as FEED_CACHE_ENTRY store (của 1 follower)

    Author->>App: Sửa bài P1 lần 1 (t1)
    App->>DB: POST(P1).version = 2
    App->>Queue: Enqueue FEED_INVALIDATION_EVENT(post=P1, event_type=edit, version=2)

    Author->>App: Sửa tiếp bài P1 lần 2 ngay sau đó (t2)
    App->>DB: POST(P1).version = 3
    App->>Queue: Enqueue FEED_INVALIDATION_EVENT(post=P1, event_type=edit, version=3)

    Note over Queue: 2 event được xử lý bởi 2 worker/node khác nhau, tới FeedCache không đúng thứ tự thực tế

    Queue->>FeedCache: Worker B xử lý trước — event version=3 tới nơi
    FeedCache->>FeedCache: post_version hiện tại (chưa có) < 3 → áp dụng, set post_version=3

    Queue->>FeedCache: Worker A xử lý version=2 tới sau (bị delay)
    FeedCache->>FeedCache: So sánh: event.version=2 <= post_version đang lưu=3
    FeedCache->>FeedCache: Bỏ qua event cũ, KHÔNG ghi đè xuống version=2

    Note over FeedCache: Nhờ so sánh version, cache feed của follower vẫn giữ đúng nội dung mới nhất (version 3), không bị event cũ tới muộn làm hỏng
```

---

# Enhance sequence — Profile change propagation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có quyết định rõ ràng nào cho trường hợp này). Đáp ứng yêu cầu thứ 4 của đề bài: khi đổi avatar/tên hiển thị, quyết định rõ chấp nhận staleness hay bắt buộc force invalidate toàn bộ snapshot cũ trong cache feed.

```mermaid
sequenceDiagram
    actor User
    participant App as Social App
    participant Policy as PROFILE_CHANGE_POLICY store
    participant DB as USER store
    participant FeedCache as FEED_CACHE_ENTRY store (rải rác ở nhiều follower)

    User->>App: Đổi avatar hoặc tên hiển thị
    App->>DB: Cập nhật USER(avatar/name)
    App->>Policy: Tra PROFILE_CHANGE_POLICY theo field vừa đổi

    alt Policy = accept_staleness (vd cho avatar)
        Policy-->>App: accept_staleness
        App-->>User: "Cập nhật thành công" — không đụng tới FEED_CACHE_ENTRY hiện có
        Note over FeedCache: Các bài cũ trong feed follower vẫn hiển thị avatar/tên cũ tạm thời, chấp nhận đánh đổi để tránh fan-out cực lớn
    else Policy = force_invalidate (vd cho display_name vì lý do định danh/tin cậy)
        Policy-->>App: force_invalidate
        App->>FeedCache: Enqueue job cập nhật author_snapshot_name/avatar trên toàn bộ FEED_CACHE_ENTRY của user này
        loop Với mỗi entry bị ảnh hưởng
            FeedCache->>FeedCache: Cập nhật author_snapshot theo giá trị mới nhất
        end
        App-->>User: "Cập nhật thành công, đang đồng bộ lại trên các feed liên quan"
        Note over FeedCache: Chi phí fan-out lớn nhưng được chấp nhận có chủ đích cho field này
    end
```
