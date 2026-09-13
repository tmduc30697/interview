# Enhance sequence — Phát hiện xung đột khi lưu (version đã lỗi thời)

Đây là **enhance**, flow mới mô tả đúng kịch bản nêu trong yêu cầu 2 của đề bài: Editor A và B cùng mở bài ở version 5, B lưu trước (bài chuyển version 6), A lưu sau vẫn gửi version 5 — request của A phải bị từ chối rõ ràng, kèm hiển thị nội dung mới nhất (version 6), không tự động ghi đè.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    actor EditorB as Editor B
    participant CMS as CMS Service
    participant DB as ARTICLE table

    EditorA->>CMS: Mở bài viết X (version = 5)
    EditorB->>CMS: Mở cùng bài viết X (version = 5)

    EditorB->>CMS: Lưu bài (gửi kèm version = 5)
    CMS->>DB: UPDATE ARTICLE SET content = ..., version = 6 WHERE id = X AND version = 5
    DB-->>CMS: 1 row affected, thành công
    CMS-->>EditorB: Lưu thành công, version mới = 6

    EditorA->>CMS: Lưu bài (vẫn gửi kèm version = 5 đã đọc từ đầu)
    CMS->>DB: UPDATE ARTICLE SET content = ..., version = 6 WHERE id = X AND version = 5
    DB-->>CMS: 0 row affected (version hiện tại trong DB đã là 6, không còn là 5)
    CMS->>DB: SELECT content, version FROM ARTICLE WHERE id = X
    DB-->>CMS: content mới nhất, version = 6
    CMS-->>EditorA: Từ chối lưu, báo "bài viết đã được người khác cập nhật", kèm nội dung version 6 mới nhất
    Note over EditorA,CMS: A phải tự quyết định merge tay hoặc ghi đè có ý thức, hệ thống không tự động merge ngầm
```
