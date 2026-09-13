# Enhance sequence — Concurrent publish/edit ordering

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có cơ chế ordering nào). Đáp ứng yêu cầu thứ 3 của đề bài: 2 invalidation event cho cùng 1 bài tới cache theo thứ tự khác thứ tự thực tế (do khác hàng đợi/khác node xử lý), phải dùng version để tránh event cũ đè nhầm lên update mới.

```mermaid
sequenceDiagram
    actor Author
    participant App as Social App
    participant DB as POST store
    participant Queue as Invalidation Queue (nhiều worker/node)
    participant FeedCache as FEED_CACHE_ENTRY store (của 1 follower)

    Author->>App: Sửa bài P1 lần 1 (t1)
    App->>DB: POST(P1).version = 2
    App->>Queue: Enqueue FEED_INVALIDATION_EVENT(post=P1, event_type=edit, version=2)

    Author->>App: Sửa tiếp bài P1 lần 2 ngay sau đó (t2)
    App->>DB: POST(P1).version = 3
    App->>Queue: Enqueue FEED_INVALIDATION_EVENT(post=P1, event_type=edit, version=3)

    Note over Queue: 2 event được xử lý bởi 2 worker/node khác nhau, tới FeedCache không đúng thứ tự thực tế

    Queue->>FeedCache: Worker B xử lý trước — event version=3 tới nơi
    FeedCache->>FeedCache: post_version hiện tại (chưa có) < 3 → áp dụng, set post_version=3

    Queue->>FeedCache: Worker A xử lý version=2 tới sau (bị delay)
    FeedCache->>FeedCache: So sánh: event.version=2 <= post_version đang lưu=3
    FeedCache->>FeedCache: Bỏ qua event cũ, KHÔNG ghi đè xuống version=2

    Note over FeedCache: Nhờ so sánh version, cache feed của follower vẫn giữ đúng nội dung mới nhất (version 3), không bị event cũ tới muộn làm hỏng
```
