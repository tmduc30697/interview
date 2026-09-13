# Enhance sequence — Publish post (fan-out-on-write vs fan-out-on-read)

Đây là **enhance**, cùng flow "Publish post" đã có ở base nhưng nay thay đổi theo yêu cầu thứ 1 của đề bài: phân biệt chiến lược theo follower_count thay vì fan-out đồng loạt cho tất cả.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant Job as FEED_FANOUT_JOB store
    participant Follow as FOLLOW store
    participant FeedCache as FEED_CACHE_ENTRY store

    Author->>App: Đăng bài mới
    App->>DB: Tạo POST(author_id, content, version=1)
    App->>App: Kiểm tra USER.follower_count của Author

    alt follower_count dưới ngưỡng (tài khoản thường)
        App->>Job: Tạo FEED_FANOUT_JOB(strategy=fanout_on_write)
        Job->>Follow: Lấy danh sách follower (số lượng nhỏ)
        Follow-->>Job: Danh sách follower
        loop Với mỗi follower
            Job->>FeedCache: Push FEED_CACHE_ENTRY(follower_id, post_id, post_version=1, author_snapshot)
        end
        Job->>Job: status=completed
    else follower_count vượt ngưỡng (celebrity account)
        App->>Job: Tạo FEED_FANOUT_JOB(strategy=fanout_on_read)
        Note over Job,FeedCache: Không push entry cho hàng triệu follower — chỉ đánh dấu bài này thuộc diện fanout_on_read
        Job->>Job: status=indexed_for_read (feed của follower sẽ tự tính bài này khi họ load feed, xem thêm cơ chế đọc feed)
    end

    App->>App: Publish FEED_INVALIDATION_EVENT (event_type=publish, version=1) — dùng cho ordering ở các flow khác
    App-->>Author: "Đăng bài thành công"
```
