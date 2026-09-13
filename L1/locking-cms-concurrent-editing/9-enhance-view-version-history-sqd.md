# Enhance sequence — Xem lịch sử version và khôi phục bản cũ

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 5 của đề bài: biên tập viên có thể xem lại lịch sử các version bài viết (ai sửa, khi nào, thay đổi gì) và khôi phục 1 version cũ nếu phát hiện bị lưu đè nhầm.

```mermaid
sequenceDiagram
    actor Editor
    participant CMS as CMS Service
    participant History as ARTICLE_VERSION_HISTORY
    participant DB as ARTICLE table

    Editor->>CMS: Xem lịch sử version của bài viết X
    CMS->>History: SELECT * FROM ARTICLE_VERSION_HISTORY WHERE article_id = X ORDER BY version DESC
    History-->>CMS: Danh sách version kèm ai sửa, khi nào, nội dung thay đổi
    CMS-->>Editor: Hiển thị timeline lịch sử version

    Editor->>CMS: Chọn khôi phục version 5 (nghi ngờ bị lưu đè nhầm ở version 7)
    CMS->>History: Lấy content_snapshot của version 5
    History-->>CMS: content_snapshot version 5
    CMS->>DB: SELECT version FROM ARTICLE WHERE id = X
    DB-->>CMS: version hiện tại = 7
    CMS->>DB: UPDATE ARTICLE SET content = <snapshot version 5>, version = 8 WHERE id = X AND version = 7
    DB-->>CMS: 1 row affected
    CMS->>History: Ghi ARTICLE_VERSION_HISTORY (version=8, change_summary="khôi phục nội dung từ version 5")
    CMS-->>Editor: Khôi phục thành công, bài viết giờ ở version 8 với nội dung của version 5
    Note over CMS,History: Việc khôi phục cũng đi qua optimistic lock như 1 lần lưu bình thường, và bản thân nó cũng được ghi vào lịch sử
```
