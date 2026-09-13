# Base sequence — Lưu bài viết (ghi đè mù, mất dữ liệu của người lưu trước)

Đây là **base**, flow lưu bài viết ở trạng thái hiện tại: không kiểm tra version, ai lưu sau sẽ ghi đè toàn bộ `content` của người lưu trước, kể cả khi 2 người sửa 2 phần nội dung khác nhau. Đây chính là vấn đề "lost update" mà toàn bộ đề bài (optimistic lock theo version) phải giải quyết.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    actor EditorB as Editor B
    participant CMS as CMS Service
    participant DB as ARTICLE table

    EditorB->>CMS: Lưu bài viết X với nội dung đã sửa
    CMS->>DB: UPDATE ARTICLE SET content = ... WHERE id = X
    DB-->>CMS: OK
    CMS-->>EditorB: Lưu thành công

    EditorA->>CMS: Lưu bài viết X với nội dung đã sửa (dựa trên bản cũ trước khi B lưu)
    CMS->>DB: UPDATE ARTICLE SET content = ... WHERE id = X
    DB-->>CMS: OK (ghi đè lên bản B vừa lưu, không phát hiện xung đột)
    CMS-->>EditorA: Lưu thành công
    Note over DB,EditorA: Toàn bộ thay đổi của Editor B đã bị mất mà không ai được cảnh báo, kể cả khi A và B sửa 2 đoạn khác nhau
```
