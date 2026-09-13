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
