# Base sequence — Load feed infinite scroll

Đây là **base**, flow "Tải feed dạng infinite scroll" ở trạng thái hiện tại — mỗi lần cần thêm bài viết, client luôn gọi API lấy trang tiếp theo, không lưu lại danh sách đã load ở đâu cả. Flow này là tiền đề cho enhance vì yêu cầu 1 của đề bài chính là bắt đầu lưu lại đúng những gì flow này đã tải.

```mermaid
sequenceDiagram
    actor User
    participant App as Feed App (tab)
    participant API as Feed API

    User->>App: Mở trang feed
    App->>API: GET /feed?cursor=null
    API-->>App: 20 bài viết đầu tiên
    App-->>User: Render feed, scroll_top = 0

    User->>App: Scroll xuống
    App->>API: GET /feed?cursor=<next>
    API-->>App: 20 bài viết tiếp theo
    App-->>User: Render nối thêm vào feed

    Note over App: Danh sách bài đã load và scroll position chỉ tồn tại trong bộ nhớ JS, không lưu lại ở đâu
```
