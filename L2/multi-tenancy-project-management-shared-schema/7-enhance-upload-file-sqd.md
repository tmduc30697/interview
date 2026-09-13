# Enhance sequence — Upload file

Đây là **enhance**, flow "Tải file đính kèm lên project". So với base, storage key không còn định danh chỉ bằng `attachment_id` mà được namespace theo `tenant_id` ngay trong đường dẫn lưu trữ, và signed URL sinh ra gắn với claim tenant — biết đúng file id nhưng sai tenant vẫn không truy cập được.

```mermaid
sequenceDiagram
    actor User
    participant WebApp
    participant API as API Server
    participant Storage as File Storage

    User->>WebApp: Tải file lên 1 project
    WebApp->>API: POST /api/projects/:projectId/files
    API->>API: Kiểm tra project thuộc đúng tenant (qua query đã scoped theo tenant)
    API->>Storage: Lưu file tại /tenants/{tenant_id}/projects/{project_id}/{file_id}/{file_name}
    Note over API,Storage: Storage key namespace theo tenant_id, không chỉ dựa vào attachment_id
    API->>API: Sinh signed URL gắn kèm claim tenant_id
    Storage-->>API: signed URL scoped đúng path của tenant
    API-->>WebApp: 201 tạo attachment, trả về signed URL đã scoped tenant
    WebApp-->>User: Hiển thị file đã tải lên
    Note over API,Storage: Request dùng signed URL với claim tenant không khớp path sẽ bị từ chối
```
