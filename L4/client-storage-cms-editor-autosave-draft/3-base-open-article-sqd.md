# Base sequence — Open article (chưa có kiểm tra draft cục bộ)

Đây là **base**, flow mở bài viết để soạn tiếp. Trình soạn thảo chỉ đơn giản tải bản mới nhất từ server và hiển thị, không có khái niệm draft cục bộ nên không có bước so sánh hay hỏi người dùng chọn phiên bản nào. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi việc mở bài viết phải kiểm tra thêm draft trong IndexedDB.

```mermaid
sequenceDiagram
    actor User as Người soạn bài
    participant Editor as Rich-text Editor (browser)
    participant API as Article API
    participant DB as Server DB

    User->>Editor: Mở bài viết để soạn tiếp
    Editor->>API: GET article theo id
    API->>DB: Lấy bản ghi ARTICLE mới nhất
    DB-->>API: Trả về content, updated_at
    API-->>Editor: Trả về nội dung bài viết
    Editor-->>User: Hiển thị nội dung để soạn tiếp
```
