# Base ERD — SSO-only B2B SaaS trước khi có cơ chế khôi phục

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có yêu cầu khôi phục quyền truy cập cấp tổ chức. Đề bài nói tài khoản "SSO-only, không có mật khẩu riêng" và mỗi công ty khách hàng dùng 1 IdP để đăng nhập — nên base chỉ cần đủ 4 entity nền tảng để việc "IdP hỏng → cả tổ chức bị khóa" có ý nghĩa: tổ chức, cấu hình IdP (đúng 1 cấu hình đang active mỗi tổ chức), user thuộc tổ chức, và session đăng nhập. Chưa có bất kỳ entity nào phục vụ khôi phục/break-glass/cutover — những thứ đó là phần enhance.

```mermaid
erDiagram
    ORGANIZATION ||--o{ USER : employs
    ORGANIZATION ||--|| IDP_CONFIG : "configured with (1 active)"
    USER ||--o{ SESSION : creates

    ORGANIZATION {
        string id PK
        string name
        string status
    }
    IDP_CONFIG {
        string id PK
        string org_id FK
        string provider_type
        string metadata_url
        string signing_cert
        string status "active"
    }
    USER {
        string id PK
        string org_id FK
        string email
        string role
        string external_idp_subject_id
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
```
