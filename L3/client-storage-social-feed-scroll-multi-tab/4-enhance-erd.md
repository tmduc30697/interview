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
