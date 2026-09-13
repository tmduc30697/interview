# Enhance ERD — Siết isolation, thêm impersonation có kiểm soát và audit log

Đây là ERD **sau khi** enhance được áp dụng lên base. Các entity nghiệp vụ (Tenant, User, Project, Task, File, Notification Config) giữ nguyên cấu trúc — điểm khác biệt nằm ở việc `tenant_id` trên các bảng này nay được enforce ở tầng thấp nhất (middleware/ORM tự động filter hoặc row-level security ở database) thay vì trông chờ code thủ công. So với base, có 2 nhóm entity mới, ứng trực tiếp với 2 yêu cầu còn lại trong đề bài:

- `IMPERSONATION_SESSION` (mới) — ghi nhận phiên nhân viên support/admin nội bộ "xem hộ" dữ liệu của 1 tenant cụ thể, có lý do, thời gian bắt đầu/kết thúc, thay vì dùng chung 1 quyền truy cập không giới hạn tenant.
- `AUDIT_LOG` (mới) — ghi log đầy đủ mọi hành động xảy ra trong lúc impersonate, cũng như các thao tác nhạy cảm khác, có thể truy vết về đúng actor và tenant liên quan.

`FILE_ATTACHMENT.storage_path` cũng đổi từ định danh theo `attachment_id` sang `storage_key` namespace theo `tenant_id`, và có thêm hạn dùng cho signed URL — khớp với yêu cầu "đường dẫn lưu trữ và khóa truy cập phải gắn chặt với tenant".

```mermaid
erDiagram
    TENANT ||--o{ USER : "có thành viên"
    TENANT ||--o{ PROJECT : "sở hữu"
    TENANT ||--o{ NOTIFICATION_CONFIG : "cấu hình"
    PROJECT ||--o{ TASK : "chứa"
    PROJECT ||--o{ FILE_ATTACHMENT : "đính kèm"
    USER ||--o{ TASK : "được giao"
    USER ||--o{ FILE_ATTACHMENT : "tải lên"
    TENANT ||--o{ IMPERSONATION_SESSION : "là đối tượng được xem hộ"
    USER ||--o{ IMPERSONATION_SESSION : "admin khởi tạo"
    TENANT ||--o{ AUDIT_LOG : "phạm vi tenant"
    USER ||--o{ AUDIT_LOG : "actor thực hiện"
    IMPERSONATION_SESSION ||--o{ AUDIT_LOG : "phát sinh"

    TENANT {
        string id PK
        string name
        string plan
    }
    USER {
        string id PK
        string tenant_id FK
        string email
        string password_hash
        string role "gồm cả internal_support"
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
        string storage_key "namespace theo tenant_id, vd /tenants/{tenant_id}/..."
        datetime signed_url_expires_at
    }
    NOTIFICATION_CONFIG {
        string id PK
        string tenant_id FK
        string channel
        string webhook_url
    }
    IMPERSONATION_SESSION {
        string id PK
        string admin_user_id FK
        string target_tenant_id FK
        string reason
        datetime started_at
        datetime ended_at
        string status "active | ended"
    }
    AUDIT_LOG {
        string id PK
        string tenant_id FK
        string actor_user_id FK
        string impersonation_session_id FK "nullable"
        string action
        string resource_type
        string resource_id
        datetime created_at
    }
```
