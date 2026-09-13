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
