# Enhance sequence — Phát hiện post đã bị xóa/chuyển private khi khôi phục từ cache

Đây là **enhance**, mở rộng thêm cho flow khôi phục ở `5-enhance-view-post-and-back-sqd.md` — trường hợp riêng khi 1 trong 50 post nằm trong `FEED_SCROLL_CACHE` đã bị tác giả xóa hoặc chuyển private trong lúc user đang ở trang khác. Trước khi render lại từ cache, App phải xác thực lại các post_id còn hiệu lực và loại bỏ post không còn quyền xem — đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant Detail as Post Detail Page
    participant App as Feed App (tab)
    participant Session as sessionStorage (FEED_SCROLL_CACHE)
    participant API as Feed API

    User->>Detail: Bấm Back (post #12 trong cache đã bị tác giả xóa lúc user đang xem bài khác)
    Detail->>App: Quay lại route feed
    App->>Session: Đọc FEED_SCROLL_CACHE (50 CACHED_POST_ITEM, bao gồm post #12)
    Session-->>App: Danh sách post_id đã cache

    App->>API: POST /posts/validate-visibility (ids = 50 post_id)
    API-->>App: post #12 = deleted, post #30 = private (không còn quyền xem), còn lại visible

    App->>App: Loại post #12 và #30 khỏi danh sách khôi phục
    App->>App: Ghi RESTORE_METRIC(restore_source=cache_hit, removed_post_count=2)
    App-->>User: Render 48 bài còn lại đúng vị trí scroll tương đối, không hiển thị post đã mất quyền xem

    Note over App: Việc validate chỉ hỏi trạng thái visibility, không tải lại nội dung toàn bộ 50 bài, giữ chi phí thấp
```
