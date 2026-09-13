# Sequence Diagram — Enhance: Publish Article

Đây là **enhance**, flow "chỉnh sửa/xuất bản bài viết" đã tồn tại ở base ([3-base-publish-article-sqd.md](3-base-publish-article-sqd.md)) nay thay đổi: mỗi thay đổi phát sinh cập nhật `ARTICLE_INDEX_DOCUMENT` gần như ngay lập tức, và bài nháp/bị gỡ do lỗi nghiêm trọng không bao giờ lọt vào index kể cả trong khoảng thời gian ngắn xử lý.

```mermaid
sequenceDiagram
    actor DocsTeam as Đội Docs
    participant HelpCenter as Help Center App
    participant DB as Articles Database
    participant Embedder as Embedding Service
    participant Indexer as Index Sync Worker
    participant Index as Article Search Index

    DocsTeam->>HelpCenter: Sửa nội dung sai/lỗi thời, lưu bản published
    HelpCenter->>DB: UPDATE articles SET content, updated_at, status
    DB-->>HelpCenter: Updated
    HelpCenter->>Indexer: Trigger sync ngay lập tức

    alt Bài viết status = published
        Indexer->>Embedder: Tạo lại embedding cho nội dung mới
        Embedder-->>Indexer: Vector mới
        Indexer->>Index: Update article_index_document (content, vector)
        Index-->>Indexer: Indexed
    else Bài viết đang ở trạng thái draft hoặc bị gỡ do lỗi nghiêm trọng
        Indexer->>Index: Đảm bảo document bị xóa/không tồn tại trong index
        Index-->>Indexer: Removed hoặc không được ghi
    end

    Note over Index: Không có khoảng trễ nào mà bài draft/removed lọt vào index, kể cả ngay sau lúc tác giả bấm lưu
    HelpCenter-->>DocsTeam: Đã lưu, nội dung mới phản ánh gần như ngay trong tìm kiếm
```
