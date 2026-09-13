# Base sequence — Upload file

Đây là **base**, flow "Tải file đính kèm lên project" — đại diện cho nhóm tài nguyên dùng chung giữa nhiều tenant nhưng cần cấu hình riêng mà đề bài nhắc tới. Chọn flow này vì storage path/URL truy cập file ở base chỉ định danh bằng `attachment_id`, chưa gắn chặt với tenant — nếu ID này đoán được thì có nguy cơ truy cập chéo tenant.

```mermaid
sequenceDiagram
    actor User
    participant WebApp
    participant API as API Server
    participant Storage as File Storage

    User->>WebApp: Tải file lên 1 project
    WebApp->>API: POST /api/projects/:projectId/files
    API->>API: Kiểm tra project thuộc đúng tenant của caller
    API->>Storage: Lưu file tại /files/{attachment_id}/{file_name}
    Note over API,Storage: Đường dẫn chỉ định danh theo attachment_id, không namespace theo tenant_id
    Storage-->>API: URL truy cập file
    API-->>WebApp: 201 tạo attachment, trả về URL tải file
    WebApp-->>User: Hiển thị file đã tải lên
    Note over Storage: Nếu ID đoán được, có nguy cơ truy cập nhầm file của tenant khác
```
