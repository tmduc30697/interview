# Base ERD — SaaS quản lý dự án multi-tenant, shared schema chưa được siết isolation

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài. Đề bài mô tả rõ hệ thống đã theo mô hình shared schema — mọi bảng đều có sẵn cột `tenant_id` để phân biệt dữ liệu giữa các tenant. Vì vậy base **đã có** các entity cốt lõi của 1 SaaS quản lý dự án (Tenant, User, Project, Task, File đính kèm, cấu hình thông báo) với cột `tenant_id` trên từng bảng — nhưng cột này mới chỉ tồn tại ở tầng dữ liệu, **chưa có** cơ chế nào đảm bảo mọi query đều tự động lọc đúng theo nó (không có middleware/ORM enforce, không có row-level security, không có test tự động, không có cơ chế impersonation có kiểm soát, background job cũng chưa chắc tôn trọng ranh giới tenant). Đó chính là khoảng trống mà enhance lấp vào.

```mermaid
erDiagram
    TENANT ||--o{ USER : "có thành viên"
    TENANT ||--o{ PROJECT : "sở hữu"
    TENANT ||--o{ NOTIFICATION_CONFIG : "cấu hình"
    PROJECT ||--o{ TASK : "chứa"
    PROJECT ||--o{ FILE_ATTACHMENT : "đính kèm"
    USER ||--o{ TASK : "được giao"
    USER ||--o{ FILE_ATTACHMENT : "tải lên"

    TENANT {
        string id PK
        string name
        string plan
    }
    USER {
        string id PK
        string tenant_id FK "cột tenant_id có sẵn trên mọi bảng"
        string email
        string password_hash
        string role
    }
    PROJECT {
        string id PK
        string tenant_id FK
        string name
        string status
    }
    TASK {
        string id PK
        string tenant_id FK
        string project_id FK
        string assignee_id FK
        string title
        string status
    }
    FILE_ATTACHMENT {
        string id PK
        string tenant_id FK
        string project_id FK
        string uploaded_by FK
        string file_name
        string storage_path "chưa chắc namespace theo tenant"
    }
    NOTIFICATION_CONFIG {
        string id PK
        string tenant_id FK
        string channel
        string webhook_url
    }
```
