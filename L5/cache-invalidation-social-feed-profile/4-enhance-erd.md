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
