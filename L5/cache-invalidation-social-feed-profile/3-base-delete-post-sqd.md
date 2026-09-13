# Base sequence — Delete post (invalidate không đáng tin cậy)

Đây là **base**, flow "Xóa bài viết" ở trạng thái hiện tại — chỉ đụng tới cache của follower đang hoạt động, không chủ động lan tới toàn bộ. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài chính là sửa đúng lỗ hổng "bài đã xóa vẫn hiển thị dai dẳng" ở đây.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant FeedCache as FEED_CACHE_ENTRY store
    actor ActiveFollower as Follower đang hoạt động
    actor InactiveFollower as Follower ít hoạt động

    Author->>App: Xóa bài viết
    App->>DB: Cập nhật POST.deleted_at
    App->>FeedCache: Cố gắng invalidate FEED_CACHE_ENTRY, nhưng chỉ chạm tới cache của follower đang active gần đây
    FeedCache-->>ActiveFollower: Cache được refresh, bài đã xóa biến mất
    Note over FeedCache,InactiveFollower: Follower ít hoạt động không được đụng tới — FEED_CACHE_ENTRY của họ vẫn còn bài đã xóa, không có cơ chế quét/lan truyền chủ động nào khác
    App-->>Author: "Xóa thành công"
```
