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
