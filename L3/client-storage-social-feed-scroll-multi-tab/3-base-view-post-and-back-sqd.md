# Base sequence — Xem chi tiết bài viết rồi bấm Back

Đây là **base**, flow "Click vào 1 bài viết rồi bấm Back" ở trạng thái hiện tại — khi quay lại, trang feed bị tải lại hoàn toàn từ đầu, mất scroll position và phải gọi lại API cho toàn bộ bài đã xem trước đó. Flow này chính là vấn đề mà yêu cầu 1 của đề bài muốn giải quyết.

```mermaid
sequenceDiagram
    actor User
    participant App as Feed App (tab)
    participant Detail as Post Detail Page
    participant API as Feed API

    User->>App: Đã scroll qua 80 bài, click vào bài #45
    App->>Detail: Điều hướng sang trang chi tiết bài #45
    Detail->>API: GET /post/45
    API-->>Detail: Nội dung bài #45
    Detail-->>User: Render chi tiết

    User->>Detail: Bấm Back
    Detail->>App: Quay lại route feed
    App->>API: GET /feed?cursor=null
    API-->>App: 20 bài viết đầu tiên (từ đầu)
    App-->>User: Render feed, scroll_top = 0

    Note over App,User: Toàn bộ 80 bài và vị trí scroll trước đó bị mất, user phải scroll lại từ đầu để tìm đúng chỗ cũ
```
