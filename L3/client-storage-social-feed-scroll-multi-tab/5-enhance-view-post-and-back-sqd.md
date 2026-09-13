# Enhance sequence — Xem chi tiết bài viết rồi bấm Back (khôi phục từ cache)

Đây là **enhance** của flow đã có ở base (`3-base-view-post-and-back-sqd.md`). So với base, trước khi rời trang feed, App lưu `FEED_SCROLL_CACHE` + tối đa 50 `CACHED_POST_ITEM` gần vị trí scroll nhất vào sessionStorage, khi Back sẽ đọc lại cache thay vì gọi API từ đầu, và ghi nhận `RESTORE_METRIC` để đo tỷ lệ cache-hit — đáp ứng yêu cầu 1 và yêu cầu 5 (đo lường) của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Feed App (tab)
    participant Session as sessionStorage (FEED_SCROLL_CACHE)
    participant Detail as Post Detail Page
    participant API as Feed API

    User->>App: Đã scroll qua 80 bài, click vào bài #45
    App->>Session: Lưu FEED_SCROLL_CACHE (scroll_top, 50 bài gần vị trí đọc nhất)
    Note over App,Session: Chỉ giữ 50 CACHED_POST_ITEM gần nhất để tránh sessionStorage phình to
    App->>Detail: Điều hướng sang trang chi tiết bài #45
    Detail->>API: GET /post/45
    API-->>Detail: Nội dung bài #45
    Detail-->>User: Render chi tiết

    User->>Detail: Bấm Back
    Detail->>App: Quay lại route feed
    App->>Session: Đọc FEED_SCROLL_CACHE của tab hiện tại
    alt Cache còn hợp lệ (chưa hết hạn, chưa vượt ngưỡng stale)
        Session-->>App: 50 CACHED_POST_ITEM + scroll_top
        App-->>User: Render lại đúng danh sách và scroll đến scroll_top
        App->>App: Ghi RESTORE_METRIC(restore_source=cache_hit)
    else Cache miss hoặc dữ liệu đã stale quá ngưỡng
        App->>API: GET /feed?cursor=null
        API-->>App: 20 bài viết đầu tiên
        App-->>User: Render feed, scroll_top = 0
        App->>App: Ghi RESTORE_METRIC(restore_source=cache_miss hoặc stale_expired)
    end
```
