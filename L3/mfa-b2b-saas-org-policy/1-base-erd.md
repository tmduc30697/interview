# Base ERD — SaaS multi-tenant chưa có chính sách MFA theo tổ chức

Đây là **base**: trạng thái SaaS quản lý dự án/CRM *trước khi* org admin có thể tự cấu hình chính sách MFA. Suy luận từ đề bài, base đã có `ORGANIZATION` (tenant), `USER` có thể tham gia nhiều tổ chức qua `ORG_MEMBERSHIP` (kèm `role`), và `SESSION` khi user đăng nhập. MFA ở base chỉ tồn tại dưới dạng tùy chọn cá nhân (`MFA_ENROLLMENT`, user tự bật nếu muốn), hoàn toàn không có khái niệm chính sách bắt buộc theo tổ chức hay theo role.

```mermaid
erDiagram
    ORGANIZATION ||--o{ ORG_MEMBERSHIP : has
    USER ||--o{ ORG_MEMBERSHIP : "tham gia"
    USER ||--o{ MFA_ENROLLMENT : "tự chọn enroll"
    USER ||--o{ SESSION : "đăng nhập"

    ORGANIZATION {
        string id PK
        string name
    }
    USER {
        string id PK
        string email
    }
    ORG_MEMBERSHIP {
        string id PK
        string org_id FK
        string user_id FK
        string role "owner|admin|member|viewer"
    }
    MFA_ENROLLMENT {
        string id PK
        string user_id FK
        string method "totp|sms"
        datetime enrolled_at
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
```
