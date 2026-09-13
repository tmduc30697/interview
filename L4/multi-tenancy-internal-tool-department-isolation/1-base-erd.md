# Base ERD — Công cụ nội bộ trước khi có phân loại tài nguyên theo phòng ban

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** áp mô hình cách ly tinh vi theo phòng ban. Đề bài đối chiếu trực tiếp với "mô hình tenant=công ty độc lập" cứng nhắc — nên base được suy luận là một quy tắc cách ly đơn giản: mỗi tài liệu thuộc về đúng 1 phòng ban (`department_id` gán lúc tạo), nhân viên chỉ xem được tài liệu của phòng ban mình đang thuộc (`department_id` hiện tại), chưa phân biệt tài nguyên dùng chung toàn công ty hay chia sẻ liên phòng ban — mọi tài nguyên bị xử lý như nhau. Base **chưa có** cơ chế chuyển phòng ban có hiệu lực theo thời điểm, **chưa có** chia sẻ có thời hạn giữa các phòng, và **chưa** tách quyền vận hành hệ thống khỏi quyền đọc nội dung nghiệp vụ (admin IT mặc định có role riêng nhưng đề bài ngụ ý ranh giới này chưa rõ ràng) — những phần này là enhance.

```mermaid
erDiagram
    DEPARTMENT ||--o{ EMPLOYEE : "has members (hiện tại)"
    DEPARTMENT ||--o{ DOCUMENT : "owns (phòng ban tạo ra)"
    EMPLOYEE ||--o{ DOCUMENT : creates
    DOCUMENT ||--o{ APPROVAL : "goes through"
    EMPLOYEE ||--o{ APPROVAL : approves

    DEPARTMENT {
        string id PK
        string name "nhân sự | tài chính | kỹ thuật | kinh doanh"
    }
    EMPLOYEE {
        string id PK
        string name
        string email
        string department_id FK "phòng ban hiện tại, 1 người 1 phòng"
        string role "employee | admin_it"
        datetime hired_at
    }
    DOCUMENT {
        string id PK
        string title
        string content
        string department_id FK "phòng ban sở hữu, gán lúc tạo"
        string owner_employee_id FK
        datetime created_at
    }
    APPROVAL {
        string id PK
        string document_id FK
        string approver_employee_id FK
        string status "pending | approved | rejected"
        datetime decided_at
    }
```
