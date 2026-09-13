# Enhance sequence — Dọn dẹp draft cục bộ sau khi lưu server hoặc publish/xoá bài

Đây là **enhance**, flow hoàn toàn mới, đảm bảo draft trong IndexedDB không tồn đọng vô thời hạn. Ngay sau khi lưu thành công lên server, hoặc khi bài viết được publish/xoá, draft cục bộ tương ứng phải được xoá để tránh phình dung lượng IndexedDB và tránh vô tình khôi phục nhầm draft lỗi thời cho một bài đã bị xoá từ lâu. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant API as Article API
    participant IDB as IndexedDB (LOCAL_DRAFT)

    alt Lưu thành công lên server (thủ công hoặc sau khi resolve xung đột)
        Editor->>API: Lưu content lên server
        API-->>Editor: Xác nhận lưu thành công, trả về updated_at mới
        Editor->>IDB: Xoá LOCAL_DRAFT của article này vì đã đồng bộ lên server
    else User bấm Publish
        Editor->>API: Publish bài viết
        API-->>Editor: Xác nhận ARTICLE.status=published
        Editor->>IDB: Xoá LOCAL_DRAFT của article này
    else User hoặc hệ thống xoá bài viết
        Editor->>API: Xoá bài viết
        API-->>Editor: Xác nhận ARTICLE.status=deleted
        Editor->>IDB: Xoá LOCAL_DRAFT của article này, tránh khôi phục nhầm cho bài đã xoá
    end

    Note over IDB: Định kỳ cũng quét các LOCAL_DRAFT không còn ARTICLE tương ứng hoặc quá cũ, để dọn rác còn sót
```
