# Enhance sequence — Save article (optimistic lock theo version, ghi lịch sử)

Đây là **enhance** của flow `save-article` đã có ở base. So với base (ghi đè mù, không kiểm tra gì), giờ mọi request lưu phải gửi kèm `version` đã đọc, server chỉ chấp nhận nếu version hiện tại trong DB còn khớp (`UPDATE ... WHERE version = 5`), đồng thời ghi snapshot vào lịch sử. Đáp ứng trực tiếp yêu cầu 1 (trường hợp lưu thành công, không có xung đột) và tạo dữ liệu cho yêu cầu 5 (lịch sử version).

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    participant CMS as CMS Service
    participant DB as ARTICLE table
    participant History as ARTICLE_VERSION_HISTORY

    EditorA->>CMS: Mở bài viết X, nhận content + version = 5
    EditorA->>CMS: Sửa nội dung, bấm Lưu (gửi kèm version = 5 đã đọc)
    CMS->>DB: UPDATE ARTICLE SET content = ..., version = 6 WHERE id = X AND version = 5
    DB-->>CMS: 1 row affected (version hiện tại đúng là 5, update thành công)
    CMS->>History: Ghi ARTICLE_VERSION_HISTORY (version=6, content_snapshot, edited_by=A, edited_at)
    CMS-->>EditorA: Lưu thành công, version mới = 6
    Note over DB,History: Vì không ai khác lưu xen giữa, điều kiện WHERE version = 5 khớp nên update đi qua bình thường
```
