# Enhance sequence — Profile change propagation

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có quyết định rõ ràng nào cho trường hợp này). Đáp ứng yêu cầu thứ 4 của đề bài: khi đổi avatar/tên hiển thị, quyết định rõ chấp nhận staleness hay bắt buộc force invalidate toàn bộ snapshot cũ trong cache feed.

```mermaid
sequenceDiagram
    actor User
    participant App as Social App
    participant Policy as PROFILE_CHANGE_POLICY store
    participant DB as USER store
    participant FeedCache as FEED_CACHE_ENTRY store (rải rác ở nhiều follower)

    User->>App: Đổi avatar hoặc tên hiển thị
    App->>DB: Cập nhật USER(avatar/name)
    App->>Policy: Tra PROFILE_CHANGE_POLICY theo field vừa đổi

    alt Policy = accept_staleness (vd cho avatar)
        Policy-->>App: accept_staleness
        App-->>User: "Cập nhật thành công" — không đụng tới FEED_CACHE_ENTRY hiện có
        Note over FeedCache: Các bài cũ trong feed follower vẫn hiển thị avatar/tên cũ tạm thời, chấp nhận đánh đổi để tránh fan-out cực lớn
    else Policy = force_invalidate (vd cho display_name vì lý do định danh/tin cậy)
        Policy-->>App: force_invalidate
        App->>FeedCache: Enqueue job cập nhật author_snapshot_name/avatar trên toàn bộ FEED_CACHE_ENTRY của user này
        loop Với mỗi entry bị ảnh hưởng
            FeedCache->>FeedCache: Cập nhật author_snapshot theo giá trị mới nhất
        end
        App-->>User: "Cập nhật thành công, đang đồng bộ lại trên các feed liên quan"
        Note over FeedCache: Chi phí fan-out lớn nhưng được chấp nhận có chủ đích cho field này
    end
```
