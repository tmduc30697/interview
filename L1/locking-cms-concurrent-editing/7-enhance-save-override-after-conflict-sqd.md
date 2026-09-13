# Enhance sequence — Lưu đè có ý thức sau khi xem nội dung mới

Đây là **enhance**, flow tiếp nối trực tiếp sau `save-conflict-detected`, đáp ứng yêu cầu 3 của đề bài: sau khi Editor A đã xem nội dung mới nhất (version 6) của Editor B, A chọn "lưu đè có ý thức" — hành động này phải tạo version mới dựa trên version hiện tại thực (6 → 7), không dùng lại version cũ (5) đã lỗi thời.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    participant CMS as CMS Service
    participant DB as ARTICLE table
    participant History as ARTICLE_VERSION_HISTORY

    Note over EditorA: A vừa nhận thông báo conflict, đã xem nội dung version 6 của B
    EditorA->>CMS: Chọn "Lưu đè" (áp nội dung A đã soạn lên trên bản version 6 mới nhất)
    CMS->>DB: SELECT version FROM ARTICLE WHERE id = X
    DB-->>CMS: version hiện tại = 6
    CMS->>DB: UPDATE ARTICLE SET content = <nội dung A>, version = 7 WHERE id = X AND version = 6
    DB-->>CMS: 1 row affected, thành công (dựa trên version thực 6, không dùng version cũ 5)
    CMS->>History: Ghi ARTICLE_VERSION_HISTORY (version=7, content_snapshot, edited_by=A, change_summary="ghi đè có ý thức sau conflict với version 6")
    CMS-->>EditorA: Lưu đè thành công, version mới = 7
    Note over CMS,History: Vì lấy đúng version hiện tại thực (6) làm mốc, hành động lưu đè không bị optimistic lock chặn lại lần 2 và vẫn giữ được vết lịch sử ai đã ghi đè lên ai
```
