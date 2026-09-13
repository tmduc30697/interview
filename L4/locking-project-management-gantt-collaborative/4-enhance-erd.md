# Enhance ERD — Optimistic locking cấp task, order riêng cho danh sách, soft delete

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm các trường/entity sau, ứng trực tiếp với từng yêu cầu:

- `TASK.version` — đáp ứng yêu cầu 1 (optimistic lock ở cấp độ từng task, không phải cấp dự án).
- `TASK.deleted_at` (soft delete thay vì xóa cứng) — đáp ứng yêu cầu 4 (phát hiện update lên task đã bị xóa để trả lỗi rõ ràng thay vì tạo lại hoặc lỗi mơ hồ).
- Entity mới `PROJECT_TASK_ORDER` với `order_version` riêng — đáp ứng yêu cầu 3 (tách version của "thứ tự toàn bộ danh sách" ra khỏi version của từng task, để kéo-thả không làm tăng version của task và không gây conflict giả khi 2 người kéo-thả các task khác nhau).
- `TASK.percent_complete` giữ nguyên kiểu, nhưng được cập nhật qua atomic increment ở DB thay vì qua optimistic lock — đáp ứng yêu cầu 5 (giảm tỷ lệ conflict phải retry cho trường bị tranh chấp cao). Không cần thêm cột mới, chỉ đổi cách ghi (xem sequence diagram tương ứng).

```mermaid
erDiagram
    PROJECT ||--o{ TASK : contains
    PROJECT ||--o{ PROJECT_MEMBER : has
    USER ||--o{ PROJECT_MEMBER : "là thành viên"
    TASK ||--o{ TASK_ASSIGNEE : "có người phụ trách"
    USER ||--o{ TASK_ASSIGNEE : "được gán"
    PROJECT ||--|| PROJECT_TASK_ORDER : "có thứ tự task"

    PROJECT {
        string id PK
        string name
    }
    PROJECT_MEMBER {
        string id PK
        string project_id FK
        string user_id FK
        string role
    }
    USER {
        string id PK
        string name
    }
    TASK {
        string id PK
        string project_id FK
        string title
        date start_date
        date deadline
        int percent_complete "cập nhật bằng atomic increment, không qua version check"
        int version "optimistic lock riêng cho từng task"
        datetime deleted_at "soft delete, null nếu chưa xóa"
        datetime updated_at
    }
    TASK_ASSIGNEE {
        string id PK
        string task_id FK
        string user_id FK
    }
    PROJECT_TASK_ORDER {
        string project_id PK
        string ordered_task_ids "JSON array thứ tự task hiện tại"
        int order_version "version riêng cho toàn bộ thứ tự danh sách, tách khỏi TASK.version"
    }
```
