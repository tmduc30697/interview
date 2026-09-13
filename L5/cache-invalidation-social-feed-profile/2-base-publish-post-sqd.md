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
