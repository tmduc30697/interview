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
