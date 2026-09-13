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
