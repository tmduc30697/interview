# Sequence Diagram — Base: Publish Article

Đây là **base**, flow "đội docs chỉnh sửa/xuất bản bài viết" — tiền đề cho enhance: mọi thay đổi nội dung diễn ra trên `ARTICLE`, và cần có index riêng đồng bộ gần như ngay lập tức từ đây để tránh hiển thị nội dung lỗi thời hoặc bài nháp.

```mermaid
sequenceDiagram
    actor DocsTeam as Đội Docs
    participant HelpCenter as Help Center App
    participant DB as Articles Database

    DocsTeam->>HelpCenter: Sửa nội dung bài viết đã xuất bản
    HelpCenter->>DB: UPDATE articles SET content, updated_at
    DB-->>HelpCenter: Updated
    HelpCenter-->>DocsTeam: Đã lưu

    Note over DB: Tìm kiếm đọc trực tiếp từ DB này nên phản ánh gần như ngay, nhưng chưa có tách bạch draft/published rõ ràng ở tầng tìm kiếm riêng
```
