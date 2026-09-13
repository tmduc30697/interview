# Enhance sequence — Delete post (tombstone chủ động + đo staleness)

Đây là **enhance**, cùng flow "Delete post" đã có ở base nhưng nay thay đổi theo yêu cầu 2 và 5 của đề bài: chủ động quét và xoá entry ở toàn bộ follower thuộc diện fanout_on_write (không chỉ follower đang hoạt động), và đo lại staleness thực tế cho từng follower.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant FeedCache as FEED_CACHE_ENTRY store
    participant Metric as FEED_STALENESS_METRIC store

    Author->>App: Xóa bài viết
    App->>DB: Cập nhật POST.deleted_at, version += 1
    App->>App: Publish FEED_INVALIDATION_EVENT (event_type=delete, version mới)

    alt Bài thuộc diện fanout_on_write
        App->>FeedCache: Quét toàn bộ FEED_CACHE_ENTRY(post_id) đang tồn tại, không chỉ của follower đang active
        loop Với mỗi entry tìm thấy
            FeedCache->>FeedCache: Kiểm tra version event > post_version đang lưu trước khi xoá (tránh đè nhầm nếu có update mới hơn)
            FeedCache->>FeedCache: Xoá/tombstone entry
            FeedCache->>Metric: Ghi FEED_STALENESS_METRIC (triggered_at=lúc xoá, reflected_at=lúc entry bị xoá, staleness_ms)
        end
        Note over FeedCache: Follower ít hoạt động cũng được dọn ngay, không phải chờ họ tự load lại mới refresh như base
    else Bài thuộc diện fanout_on_read
        Note over FeedCache: Không có entry vật lý nào cần xoá — lần đọc feed tiếp theo của follower tự loại bỏ bài đã xóa do kiểm tra deleted_at tại thời điểm đọc
    end

    App-->>Author: "Xóa thành công"
```
