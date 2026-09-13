# Base sequence — Mở bài viết để sửa (không có khái niệm version)

Đây là **base**, flow mở bài viết hiện tại: editor mở bài, nhận toàn bộ nội dung hiện có, không có thông tin version đi kèm vì hệ thống chưa track version. Đây là tiền đề để so sánh với yêu cầu 1 (thêm cột version phải trả về khi mở bài để client gửi kèm lúc lưu).

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    actor EditorB as Editor B
    participant CMS as CMS Service
    participant DB as ARTICLE table

    EditorA->>CMS: Mở bài viết X để sửa
    CMS->>DB: SELECT content FROM ARTICLE WHERE id = X
    DB-->>CMS: content hiện tại
    CMS-->>EditorA: Trả nội dung bài viết (không có version)

    EditorB->>CMS: Mở cùng bài viết X để sửa
    CMS->>DB: SELECT content FROM ARTICLE WHERE id = X
    DB-->>CMS: content hiện tại (giống hệt A vừa đọc)
    CMS-->>EditorB: Trả nội dung bài viết (không có version)
    Note over EditorA,EditorB: Cả 2 editor đang cùng sửa trên 1 bản nội dung như nhau, không ai biết người kia cũng đang mở bài
```
