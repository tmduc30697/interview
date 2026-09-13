# Enhance sequence — List projects

Đây là **enhance**, flow "Đọc danh sách project của tenant hiện tại". So với base, điều kiện lọc `tenant_id` không còn do lập trình viên tự viết trong từng câu query — middleware tự động gán tenant context cho session, và database enforce bằng row-level security, nên dù application code có quên thêm điều kiện thì dữ liệu chéo tenant vẫn không thể lộ ra.

```mermaid
sequenceDiagram
    actor User
    participant WebApp
    participant API as API Server
    participant MW as Tenant Scoping Middleware
    participant DB as Shared Database (đã bật RLS)

    User->>WebApp: Mở trang danh sách project
    WebApp->>API: GET /api/projects (kèm session token)
    API->>MW: Xác định tenant_id từ session
    MW->>DB: SET app.current_tenant_id = :tenant_id (theo connection)
    API->>DB: SELECT * FROM projects
    Note over DB: Row-Level Security policy tự động lọc theo app.current_tenant_id
    Note over DB: Enforce ở tầng database, không phụ thuộc application code có nhớ thêm WHERE hay không
    DB-->>API: Rows đã được lọc đúng tenant
    API-->>WebApp: 200 danh sách project
    WebApp-->>User: Hiển thị danh sách
```
