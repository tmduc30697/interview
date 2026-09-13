# Base ERD — Quản lý dự án chưa có cơ chế kiểm soát tương tranh

Đây là **base**: trạng thái tool quản lý dự án dạng Gantt chart *trước khi* áp dụng optimistic locking. Suy luận từ đề bài, base đã có `PROJECT` chứa nhiều `TASK`, mỗi task có thể có nhiều người phụ trách (`TASK_ASSIGNEE`), và `PROJECT_MEMBER` xác định ai là thành viên của dự án. Task đã có `deadline`, `percent_complete`, `order_index` để vẽ Gantt chart, nhưng chưa có trường `version` hay bất kỳ cơ chế phát hiện xung đột nào — hai người sửa cùng lúc sẽ ghi đè lẫn nhau (lost update).

```mermaid
erDiagram
    PROJECT ||--o{ TASK : contains
    PROJECT ||--o{ PROJECT_MEMBER : has
    USER ||--o{ PROJECT_MEMBER : "là thành viên"
    TASK ||--o{ TASK_ASSIGNEE : "có người phụ trách"
    USER ||--o{ TASK_ASSIGNEE : "được gán"

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
        int percent_complete
        int order_index "vị trí trên Gantt chart"
        datetime updated_at
    }
    TASK_ASSIGNEE {
        string id PK
        string task_id FK
        string user_id FK
    }
```
