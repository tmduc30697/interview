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
