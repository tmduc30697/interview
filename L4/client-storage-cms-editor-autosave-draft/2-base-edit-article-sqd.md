# Base sequence — Edit article (lưu thẳng lên server, chưa có autosave cục bộ)

Đây là **base**, flow soạn thảo bài viết ở trạng thái trước enhance. Người dùng gõ nội dung, việc lưu chỉ xảy ra khi bấm nút Save thủ công hoặc autosave định kỳ gọi thẳng API server, không có bước lưu tạm cục bộ nào. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi autosave chuyển sang debounce ghi vào IndexedDB trước.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant API as Article API
    participant DB as Server DB

    User->>Editor: Gõ nội dung liên tục
    loop Autosave định kỳ mỗi X giây, không phân biệt độ dài nội dung
        Editor->>API: POST toàn bộ content hiện tại
        API->>DB: Cập nhật ARTICLE.content, updated_at
        API-->>Editor: Xác nhận đã lưu
    end
    User->>Editor: Bấm nút Save thủ công
    Editor->>API: POST content hiện tại
    API->>DB: Cập nhật ARTICLE.content, updated_at
    API-->>Editor: Xác nhận đã lưu
```
