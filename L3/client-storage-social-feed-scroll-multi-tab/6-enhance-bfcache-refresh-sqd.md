# Enhance sequence — Khôi phục từ bfcache và refresh dữ liệu stale

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — chưa tồn tại ở base vì base chưa từng quan tâm tới bfcache. Khi trình duyệt khôi phục trang từ bfcache (không chạy lại JS), App phải lắng nghe sự kiện `pageshow` với `event.persisted === true` để chủ động refresh những phần dữ liệu dễ đổi (like/comment count) thay vì hiển thị y nguyên trạng thái đóng băng — đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Browser as Browser (bfcache)
    participant App as Feed App (tab, khôi phục từ memory)
    participant API as Feed API

    User->>Browser: Bấm Back (trang trước đó đủ điều kiện vào bfcache)
    Browser-->>App: Khôi phục toàn bộ DOM + JS state từ memory, không chạy lại script
    Browser->>App: Fire event pageshow (persisted = true)

    App->>App: Kiểm tra persisted === true, tab_visible = true
    App->>API: GET /feed/refresh-counters?ids=<50 post id đang hiển thị>
    API-->>App: like_count/comment_count mới nhất cho 50 bài
    App-->>User: Cập nhật lại số like/comment hiển thị, giữ nguyên scroll_top và danh sách bài

    Note over App: Không gọi lại toàn bộ /feed, chỉ refresh phần số liệu dễ đổi, tránh flash toàn màn hình
```
