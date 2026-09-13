# Enhance ERD — sau khi có phân loại tài nguyên + chia sẻ có thời hạn + chuyển phòng ban

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm thay đổi/entity mới, ứng trực tiếp với 4 yêu cầu trong đề bài:

- `DOCUMENT` (sửa) — thêm `resource_type` phân biệt 3 loại: cách ly tuyệt đối theo phòng ban, dùng chung toàn công ty, chia sẻ có chọn lọc liên phòng ban.
- `DEPARTMENT_TRANSFER` (mới) — chuyển phòng ban có thời điểm hiệu lực rõ ràng, tách biệt với `EMPLOYEE.department_id` hiện tại; tài liệu cũ (`DOCUMENT.department_id`) không tự chuyển theo người.
- `BUSINESS_CONTENT_GRANT` (mới) — cấp quyền đọc nội dung nghiệp vụ tường minh, có audit, cho admin IT (mặc định admin IT không có quyền này, chỉ có quyền vận hành).
- `CROSS_DEPARTMENT_SHARE` + `SHARE_TARGET_DEPARTMENT` (mới) — chia sẻ tài liệu `cross_department_shared` cho một hoặc nhiều phòng ban cụ thể, có `valid_from`/`valid_until`, tự động hết hạn.

```mermaid
erDiagram
    DEPARTMENT ||--o{ EMPLOYEE : "has members (hiện tại)"
    DEPARTMENT ||--o{ DOCUMENT : "owns (phòng ban tạo ra, không đổi)"
    EMPLOYEE ||--o{ DOCUMENT : creates
    DOCUMENT ||--o{ APPROVAL : "goes through"
    EMPLOYEE ||--o{ APPROVAL : approves
    EMPLOYEE ||--o{ DEPARTMENT_TRANSFER : "undergoes"
    EMPLOYEE ||--o{ BUSINESS_CONTENT_GRANT : "granted to (thường là admin_it)"
    DEPARTMENT ||--o{ BUSINESS_CONTENT_GRANT : "scope"
    DOCUMENT ||--o{ CROSS_DEPARTMENT_SHARE : "shared via"
    CROSS_DEPARTMENT_SHARE ||--o{ SHARE_TARGET_DEPARTMENT : "targets"
    DEPARTMENT ||--o{ SHARE_TARGET_DEPARTMENT : "receives share"

    DEPARTMENT {
        string id PK
        string name "nhân sự | tài chính | kỹ thuật | kinh doanh"
    }
    EMPLOYEE {
        string id PK
        string name
        string email
        string department_id FK "phòng ban hiện tại, cập nhật khi transfer có hiệu lực"
        string role "employee | admin_it"
        datetime hired_at
    }
    DOCUMENT {
        string id PK
        string title
        string content
        string department_id FK "phòng ban sở hữu, gán lúc tạo, không đổi theo người tạo"
        string owner_employee_id FK
        string resource_type "department_only | company_wide | cross_department_shared"
        datetime created_at
    }
    APPROVAL {
        string id PK
        string document_id FK
        string approver_employee_id FK
        string status "pending | approved | rejected"
        datetime decided_at
    }
    DEPARTMENT_TRANSFER {
        string id PK
        string employee_id FK
        string from_department_id FK
        string to_department_id FK
        datetime effective_at
        datetime created_at
        string created_by
    }
    BUSINESS_CONTENT_GRANT {
        string id PK
        string employee_id FK "thường là admin_it"
        string department_id FK "phòng ban được phép đọc nội dung"
        string granted_by
        string reason
        datetime granted_at
        datetime revoked_at "nullable"
    }
    CROSS_DEPARTMENT_SHARE {
        string id PK
        string document_id FK
        string project_name
        datetime valid_from
        datetime valid_until
        string status "active | expired | revoked"
        string created_by
    }
    SHARE_TARGET_DEPARTMENT {
        string share_id FK
        string department_id FK
    }
```
