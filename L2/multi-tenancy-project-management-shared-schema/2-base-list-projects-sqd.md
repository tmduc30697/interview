# Base sequence — List projects

Đây là **base**, flow "Đọc danh sách project của tenant hiện tại" — đại diện cho mọi query đọc/ghi dữ liệu thông thường trong hệ thống. Chọn flow này vì nó là ví dụ điển hình cho vấn đề đề bài nêu: điều kiện lọc `tenant_id` được thêm thủ công ở tầng application code, phụ thuộc hoàn toàn vào việc lập trình viên nhớ viết đúng ở từng câu query — đây chính là rủi ro mà enhance phải loại bỏ.

```mermaid
sequenceDiagram
    actor User
    participant WebApp
    participant API as API Server
    participant DB as Shared Database

    User->>WebApp: Mở trang danh sách project
    WebApp->>API: GET /api/projects (kèm session token)
    API->>API: Xác định tenant_id từ session
    API->>DB: SELECT * FROM projects WHERE tenant_id = :tenant_id
    Note over API,DB: Điều kiện WHERE tenant_id do lập trình viên tự viết tay
    Note over API,DB: Chỉ cần quên thêm điều kiện này ở 1 câu query là lộ dữ liệu chéo tenant
    DB-->>API: Trả về rows (đúng nếu filter được viết đúng)
    API-->>WebApp: 200 danh sách project
    WebApp-->>User: Hiển thị danh sách
```
