# Enhance sequence — Auto-save draft riêng theo user, không tăng version chính thức

Đây là **enhance**, flow hoàn toàn mới đáp ứng yêu cầu 4 của đề bài: trong lúc Editor đang soạn, hệ thống tự động lưu draft định kỳ vào `ARTICLE_DRAFT` riêng theo từng user, hoàn toàn tách khỏi `ARTICLE.version` chính thức — không gây xung đột giả với người khác đang sửa cùng bài.

```mermaid
sequenceDiagram
    actor EditorA as Editor A
    participant Client as CMS Client (auto-save timer)
    participant CMS as CMS Service
    participant Draft as ARTICLE_DRAFT (riêng theo editor_id)
    participant DB as ARTICLE table (bản chính thức)

    EditorA->>Client: Đang soạn nội dung bài viết X
    loop Mỗi 30 giây (auto-save)
        Client->>CMS: Auto-save draft hiện tại (article_id=X, editor_id=A)
        CMS->>Draft: UPSERT ARTICLE_DRAFT (article_id=X, editor_id=A, draft_content, saved_at)
        Draft-->>CMS: OK
        Note over DB: ARTICLE.version của bản chính thức hoàn toàn không đổi
    end
    EditorA->>Client: Bấm "Publish" (lưu chính thức)
    Client->>CMS: Publish với version đã đọc ban đầu
    CMS->>DB: UPDATE ARTICLE ... WHERE version = <version đã đọc> (áp dụng optimistic lock như flow save-article)
    DB-->>CMS: Kết quả tuỳ còn khớp version hay không
    CMS-->>EditorA: Chỉ lúc publish mới có khả năng bị conflict, auto-save trước đó không tạo xung đột nào
```
