# Enhance sequence — Đồng bộ like/unlike giữa 2 tab

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — ở base mỗi tab like/unlike độc lập, không tab nào biết tab kia vừa đổi trạng thái. Enhance dùng BroadcastChannel (fallback bằng `storage` event) để phát `LIKE_SYNC_EVENT` ngay khi 1 tab like/unlike, các tab khác đang mở cùng post cập nhật UI theo — đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant TabA as Tab A (đang mở post #45)
    participant Channel as BroadcastChannel "post-likes"
    participant TabB as Tab B (đang mở cùng post #45)
    participant API as Feed API

    User->>TabA: Bấm Like bài #45
    TabA->>API: POST /post/45/like
    API-->>TabA: 200 OK, like_count = 121
    TabA-->>User: Cập nhật UI tab A, hiển thị đã like

    TabA->>Channel: postMessage(LIKE_SYNC_EVENT{post_id=45, action=like, origin=tab_a})
    Channel-->>TabB: Nhận LIKE_SYNC_EVENT
    TabB->>TabB: Kiểm tra post_id đang hiển thị có khớp không
    TabB-->>User: Cập nhật UI tab B, hiển thị đã like, like_count = 121

    Note over TabA,TabB: Không gọi lại API ở tab B, chỉ đồng bộ state qua message, tránh 2 tab lệch trạng thái like của cùng 1 user
```
