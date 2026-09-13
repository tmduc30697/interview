# ERD - Enhance (distributed tracing xuyên các service)

Đây là trạng thái **enhance**: giữ nguyên `TASK` từ base, thay `SERVICE_LOG` rời rạc bằng mô hình tracing có cấu trúc:
- `TRACE`: đại diện cho toàn bộ hành trình 1 request, có `trace_id` sinh ở gateway (hoặc nhận từ client) - đáp ứng yêu cầu 1.
- `SPAN`: mỗi service tạo 1 span con gắn `trace_id` gốc và `parent_span_id`, có `is_synthetic_passthrough` để đánh dấu đoạn đi qua service không hỗ trợ tracing (chỉ pass-through header, không có span chi tiết) - đáp ứng yêu cầu 2 và 3. Span tự chứa đủ thông tin (`trace_id`, `span_id`, `start_time`, `end_time`, `status`) để đứng độc lập, không phụ thuộc thứ tự nhận - đáp ứng yêu cầu 5.
- `SPAN_ATTRIBUTE`: cặp key/value gắn vào span, chỉ được ghi nếu `key` nằm trong `TRACE_ATTRIBUTE_ALLOWLIST` - đáp ứng yêu cầu 4.
- `TRACE_ATTRIBUTE_ALLOWLIST`: danh sách các attribute key được phép ghi vào trace, dùng để lọc trước khi ghi `SPAN_ATTRIBUTE`, ngăn rò rỉ password/token - đáp ứng yêu cầu 4.
- `MQ_MESSAGE_HEADER`: đại diện cho việc propagate `trace_id`/`parent_span_id` qua header của message queue khi gọi bất đồng bộ - đáp ứng yêu cầu 1.

```mermaid
erDiagram
    TASK ||--o{ SPAN : "được xử lý qua các span"
    TRACE ||--o{ SPAN : "gồm nhiều span con"
    SPAN ||--o{ SPAN : "parent_span_id tự tham chiếu"
    SPAN ||--o{ SPAN_ATTRIBUTE : "gắn attribute"
    TRACE_ATTRIBUTE_ALLOWLIST ||--o{ SPAN_ATTRIBUTE : "kiểm tra trước khi ghi"
    SPAN ||--o| MQ_MESSAGE_HEADER : "propagate qua message queue"

    TASK {
        string task_id PK
        string project_id
        string title
        string status
        datetime created_at
    }
    TRACE {
        string trace_id PK
        string root_service
        datetime started_at
    }
    SPAN {
        string span_id PK
        string trace_id FK
        string parent_span_id FK
        string service_name
        string operation_name
        datetime start_time
        datetime end_time
        string status
        boolean is_synthetic_passthrough
    }
    SPAN_ATTRIBUTE {
        string span_id FK
        string key
        string value
    }
    TRACE_ATTRIBUTE_ALLOWLIST {
        string key PK
        string description
    }
    MQ_MESSAGE_HEADER {
        string message_id PK
        string trace_id FK
        string parent_span_id FK
    }
```
