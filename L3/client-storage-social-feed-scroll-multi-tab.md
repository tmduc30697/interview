# Khôi phục scroll feed và đồng bộ trạng thái đa tab cho mạng xã hội

**Hệ thống:** Mạng xã hội có feed dạng infinite scroll; khi user click vào 1 bài viết rồi bấm Back, cần khôi phục đúng vị trí scroll và danh sách bài đã load, đồng thời đồng bộ trạng thái đọc/like giữa nhiều tab cùng mở.

**Vai trò của flow:** Giữ trải nghiệm mượt (không phải load lại từ đầu feed khi quay lại) mà vẫn đảm bảo dữ liệu hiển thị không quá cũ so với hành động đã thực hiện ở tab khác.

**Yêu cầu cụ thể:**
- Lưu danh sách post đã load + vị trí scroll vào sessionStorage (theo tab, theo session) khi rời trang; khi quay lại bằng Back, khôi phục đúng danh sách và scroll position — nhưng phải giới hạn kích thước lưu (ví dụ chỉ giữ 50 post gần nhất) để tránh sessionStorage phình to khi user scroll rất sâu.
- Trình duyệt có bfcache (back/forward cache) khôi phục toàn bộ trang từ memory kể cả JS state mà không chạy lại code — phải xử lý đúng sự kiện `pageshow`/`persisted` để refresh những phần dữ liệu có thể đã đổi (số like, comment mới) thay vì hiển thị y nguyên trạng thái cũ đóng băng từ lúc rời trang.
- Khi user like/unlike một post ở tab A, cần đồng bộ trạng thái đó sang tab B đang mở cùng post đó (qua BroadcastChannel hoặc `storage` event) để tránh hiển thị lệch trạng thái like giữa 2 tab của cùng 1 user.
- Nếu post trong feed đã cache bị tác giả xóa hoặc chuyển private trong lúc user đang ở trang khác, khi quay lại từ cache phải phát hiện và loại bỏ post đó khỏi danh sách khôi phục thay vì hiển thị nội dung đã không còn quyền xem.
- Đo lường: tỷ lệ lần quay lại (Back) khôi phục thành công từ cache (không phải load lại từ API), so với số lần phải fallback load mới do cache miss/hết hạn/dữ liệu đã stale quá ngưỡng cho phép.

---

# Base ERD — Mạng xã hội feed infinite scroll trước khi có khôi phục scroll/đồng bộ đa tab

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, mô tả trạng thái hệ thống **trước khi** áp enhance. Đề bài nói tới feed infinite scroll, bài viết, like/comment, và trạng thái đọc — nên base chỉ cần đủ User, Post, Like để feed hoạt động bình thường ở phía server. Chưa có bất kỳ entity/cấu trúc nào phía client (sessionStorage, cache theo tab, đồng bộ đa tab) — toàn bộ phần đó là enhance.

```mermaid
erDiagram
    USER ||--o{ POST : "authors"
    USER ||--o{ LIKE : "likes"
    POST ||--o{ LIKE : "liked by"

    USER {
        string id PK
        string display_name
    }
    POST {
        string id PK
        string author_id FK
        string content
        string visibility "public | private"
        int like_count
        int comment_count
        datetime created_at
    }
    LIKE {
        string id PK
        string user_id FK
        string post_id FK
        datetime created_at
    }
```

---

# Base sequence — Load feed infinite scroll

Đây là **base**, flow "Tải feed dạng infinite scroll" ở trạng thái hiện tại — mỗi lần cần thêm bài viết, client luôn gọi API lấy trang tiếp theo, không lưu lại danh sách đã load ở đâu cả. Flow này là tiền đề cho enhance vì yêu cầu 1 của đề bài chính là bắt đầu lưu lại đúng những gì flow này đã tải.

```mermaid
sequenceDiagram
    actor User
    participant App as Feed App (tab)
    participant API as Feed API

    User->>App: Mở trang feed
    App->>API: GET /feed?cursor=null
    API-->>App: 20 bài viết đầu tiên
    App-->>User: Render feed, scroll_top = 0

    User->>App: Scroll xuống
    App->>API: GET /feed?cursor=<next>
    API-->>App: 20 bài viết tiếp theo
    App-->>User: Render nối thêm vào feed

    Note over App: Danh sách bài đã load và scroll position chỉ tồn tại trong bộ nhớ JS, không lưu lại ở đâu
```

---

# Base sequence — Xem chi tiết bài viết rồi bấm Back

Đây là **base**, flow "Click vào 1 bài viết rồi bấm Back" ở trạng thái hiện tại — khi quay lại, trang feed bị tải lại hoàn toàn từ đầu, mất scroll position và phải gọi lại API cho toàn bộ bài đã xem trước đó. Flow này chính là vấn đề mà yêu cầu 1 của đề bài muốn giải quyết.

```mermaid
sequenceDiagram
    actor User
    participant App as Feed App (tab)
    participant Detail as Post Detail Page
    participant API as Feed API

    User->>App: Đã scroll qua 80 bài, click vào bài #45
    App->>Detail: Điều hướng sang trang chi tiết bài #45
    Detail->>API: GET /post/45
    API-->>Detail: Nội dung bài #45
    Detail-->>User: Render chi tiết

    User->>Detail: Bấm Back
    Detail->>App: Quay lại route feed
    App->>API: GET /feed?cursor=null
    API-->>App: 20 bài viết đầu tiên (từ đầu)
    App-->>User: Render feed, scroll_top = 0

    Note over App,User: Toàn bộ 80 bài và vị trí scroll trước đó bị mất, user phải scroll lại từ đầu để tìm đúng chỗ cũ
```

---

# Enhance ERD — sau khi có cache scroll theo tab và đồng bộ trạng thái đa tab

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (chỉ có USER/POST/LIKE ở server), phần enhance thêm 4 entity hoàn toàn mới ở phía client, ứng trực tiếp với các yêu cầu trong đề bài:

- `TAB_SESSION` — 1 bản ghi cho mỗi tab đang mở, gắn với sessionStorage riêng của tab đó.
- `FEED_SCROLL_CACHE` (mới) — snapshot scroll position + danh sách bài đã load, lưu trong sessionStorage, giới hạn `max_post_count` (ví dụ 50) — đáp ứng yêu cầu 1.
- `CACHED_POST_ITEM` (mới) — từng bài trong cache, kèm snapshot like/comment count tại thời điểm lưu, để phát hiện dữ liệu đã stale khi bfcache khôi phục — đáp ứng yêu cầu 2.
- `LIKE_SYNC_EVENT` (mới) — sự kiện phát qua BroadcastChannel/`storage` khi like/unlike, để tab khác cùng post đồng bộ theo — đáp ứng yêu cầu 3.
- `RESTORE_METRIC` (mới) — ghi nhận mỗi lần Back là cache-hit hay fallback (cache miss/stale/post bị gỡ), phục vụ đo lường ở yêu cầu 5; trường `removed_post_count` phục vụ yêu cầu 4.

```mermaid
erDiagram
    USER ||--o{ POST : "authors"
    USER ||--o{ LIKE : "likes"
    POST ||--o{ LIKE : "liked by"
    TAB_SESSION ||--o| FEED_SCROLL_CACHE : "holds (per tab)"
    FEED_SCROLL_CACHE ||--o{ CACHED_POST_ITEM : "contains, max 50"
    CACHED_POST_ITEM }o--|| POST : "snapshot of"
    LIKE_SYNC_EVENT }o--|| POST : "about"
    LIKE_SYNC_EVENT }o--|| USER : "triggered by"
    TAB_SESSION ||--o{ RESTORE_METRIC : "generates on Back"

    USER {
        string id PK
        string display_name
    }
    POST {
        string id PK
        string author_id FK
        string content
        string visibility "public | private"
        int like_count
        int comment_count
        datetime created_at
    }
    LIKE {
        string id PK
        string user_id FK
        string post_id FK
        datetime created_at
    }
    TAB_SESSION {
        string tab_id PK
        string browser_session_id
        datetime opened_at
    }
    FEED_SCROLL_CACHE {
        string id PK
        string tab_id FK
        int scroll_top
        int max_post_count "50"
        datetime saved_at
    }
    CACHED_POST_ITEM {
        string id PK
        string cache_id FK
        string post_id FK
        int position_order
        int like_count_snapshot
        int comment_count_snapshot
        string visibility_snapshot
    }
    LIKE_SYNC_EVENT {
        string id PK
        string post_id FK
        string user_id FK
        string action "like | unlike"
        string origin_tab_id
        datetime broadcast_at
    }
    RESTORE_METRIC {
        string id PK
        string tab_id FK
        string restore_source "cache_hit | cache_miss | stale_expired"
        int removed_post_count
        datetime at
    }
```

---

# Enhance sequence — Xem chi tiết bài viết rồi bấm Back (khôi phục từ cache)

Đây là **enhance** của flow đã có ở base (`3-base-view-post-and-back-sqd.md`). So với base, trước khi rời trang feed, App lưu `FEED_SCROLL_CACHE` + tối đa 50 `CACHED_POST_ITEM` gần vị trí scroll nhất vào sessionStorage, khi Back sẽ đọc lại cache thay vì gọi API từ đầu, và ghi nhận `RESTORE_METRIC` để đo tỷ lệ cache-hit — đáp ứng yêu cầu 1 và yêu cầu 5 (đo lường) của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Feed App (tab)
    participant Session as sessionStorage (FEED_SCROLL_CACHE)
    participant Detail as Post Detail Page
    participant API as Feed API

    User->>App: Đã scroll qua 80 bài, click vào bài #45
    App->>Session: Lưu FEED_SCROLL_CACHE (scroll_top, 50 bài gần vị trí đọc nhất)
    Note over App,Session: Chỉ giữ 50 CACHED_POST_ITEM gần nhất để tránh sessionStorage phình to
    App->>Detail: Điều hướng sang trang chi tiết bài #45
    Detail->>API: GET /post/45
    API-->>Detail: Nội dung bài #45
    Detail-->>User: Render chi tiết

    User->>Detail: Bấm Back
    Detail->>App: Quay lại route feed
    App->>Session: Đọc FEED_SCROLL_CACHE của tab hiện tại
    alt Cache còn hợp lệ (chưa hết hạn, chưa vượt ngưỡng stale)
        Session-->>App: 50 CACHED_POST_ITEM + scroll_top
        App-->>User: Render lại đúng danh sách và scroll đến scroll_top
        App->>App: Ghi RESTORE_METRIC(restore_source=cache_hit)
    else Cache miss hoặc dữ liệu đã stale quá ngưỡng
        App->>API: GET /feed?cursor=null
        API-->>App: 20 bài viết đầu tiên
        App-->>User: Render feed, scroll_top = 0
        App->>App: Ghi RESTORE_METRIC(restore_source=cache_miss hoặc stale_expired)
    end
```

---

# Enhance sequence — Khôi phục từ bfcache và refresh dữ liệu stale

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base chưa từng quan tâm tới bfcache. Khi trình duyệt khôi phục trang từ bfcache (không chạy lại JS), App phải lắng nghe sự kiện `pageshow` với `event.persisted === true` để chủ động refresh những phần dữ liệu dễ đổi (like/comment count) thay vì hiển thị y nguyên trạng thái đóng băng — đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Browser as Browser (bfcache)
    participant App as Feed App (tab, khôi phục từ memory)
    participant API as Feed API

    User->>Browser: Bấm Back (trang trước đó đủ điều kiện vào bfcache)
    Browser-->>App: Khôi phục toàn bộ DOM + JS state từ memory, không chạy lại script
    Browser->>App: Fire event pageshow (persisted = true)

    App->>App: Kiểm tra persisted === true, tab_visible = true
    App->>API: GET /feed/refresh-counters?ids=<50 post id đang hiển thị>
    API-->>App: like_count/comment_count mới nhất cho 50 bài
    App-->>User: Cập nhật lại số like/comment hiển thị, giữ nguyên scroll_top và danh sách bài

    Note over App: Không gọi lại toàn bộ /feed, chỉ refresh phần số liệu dễ đổi, tránh flash toàn màn hình
```

---

# Enhance sequence — Đồng bộ like/unlike giữa 2 tab

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — ở base mỗi tab like/unlike độc lập, không tab nào biết tab kia vừa đổi trạng thái. Enhance dùng BroadcastChannel (fallback bằng `storage` event) để phát `LIKE_SYNC_EVENT` ngay khi 1 tab like/unlike, các tab khác đang mở cùng post cập nhật UI theo — đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant TabA as Tab A (đang mở post #45)
    participant Channel as BroadcastChannel "post-likes"
    participant TabB as Tab B (đang mở cùng post #45)
    participant API as Feed API

    User->>TabA: Bấm Like bài #45
    TabA->>API: POST /post/45/like
    API-->>TabA: 200 OK, like_count = 121
    TabA-->>User: Cập nhật UI tab A, hiển thị đã like

    TabA->>Channel: postMessage(LIKE_SYNC_EVENT{post_id=45, action=like, origin=tab_a})
    Channel-->>TabB: Nhận LIKE_SYNC_EVENT
    TabB->>TabB: Kiểm tra post_id đang hiển thị có khớp không
    TabB-->>User: Cập nhật UI tab B, hiển thị đã like, like_count = 121

    Note over TabA,TabB: Không gọi lại API ở tab B, chỉ đồng bộ state qua message, tránh 2 tab lệch trạng thái like của cùng 1 user
```

---

# Enhance sequence — Phát hiện post đã bị xóa/chuyển private khi khôi phục từ cache

Đây là **enhance**, mở rộng thêm cho flow khôi phục ở `5-enhance-view-post-and-back-sqd.md` — trường hợp riêng khi 1 trong 50 post nằm trong `FEED_SCROLL_CACHE` đã bị tác giả xóa hoặc chuyển private trong lúc user đang ở trang khác. Trước khi render lại từ cache, App phải xác thực lại các post_id còn hiệu lực và loại bỏ post không còn quyền xem — đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Detail as Post Detail Page
    participant App as Feed App (tab)
    participant Session as sessionStorage (FEED_SCROLL_CACHE)
    participant API as Feed API

    User->>Detail: Bấm Back (post #12 trong cache đã bị tác giả xóa lúc user đang xem bài khác)
    Detail->>App: Quay lại route feed
    App->>Session: Đọc FEED_SCROLL_CACHE (50 CACHED_POST_ITEM, bao gồm post #12)
    Session-->>App: Danh sách post_id đã cache

    App->>API: POST /posts/validate-visibility (ids = 50 post_id)
    API-->>App: post #12 = deleted, post #30 = private (không còn quyền xem), còn lại visible

    App->>App: Loại post #12 và #30 khỏi danh sách khôi phục
    App->>App: Ghi RESTORE_METRIC(restore_source=cache_hit, removed_post_count=2)
    App-->>User: Render 48 bài còn lại đúng vị trí scroll tương đối, không hiển thị post đã mất quyền xem

    Note over App: Việc validate chỉ hỏi trạng thái visibility, không tải lại nội dung toàn bộ 50 bài, giữ chi phí thấp
```
