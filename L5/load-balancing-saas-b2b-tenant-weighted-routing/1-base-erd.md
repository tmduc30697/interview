# Base ERD — API gateway chưa có giới hạn/tách biệt tài nguyên theo tenant

Đây là **base**: trạng thái API gateway *trước khi* áp dụng giới hạn tài nguyên và routing có trọng số theo tenant. Suy luận từ đề bài, base đã có `TENANT` (công ty khách hàng, xác định qua API key/JWT khi gọi request), nhiều `GATEWAY_INSTANCE` chạy song song đứng trước `BACKEND_INSTANCE`, và gateway chọn backend bằng thuật toán đơn giản (round-robin/least-connections) không quan tâm tenant nào đang gửi request. Base chưa có bất kỳ khái niệm quota, bộ đếm chia sẻ, hay trọng số theo gói dịch vụ nào.

```mermaid
erDiagram
    TENANT ||--o{ REQUEST_LOG : "gửi"
    GATEWAY_INSTANCE ||--o{ REQUEST_LOG : "xử lý"
    BACKEND_INSTANCE ||--o{ REQUEST_LOG : "phục vụ"

    TENANT {
        string id PK
        string name
        string plan_tier "free|standard|premium (chưa dùng để routing)"
        string api_key
    }
    GATEWAY_INSTANCE {
        string id PK
        string host
        string status "active|down"
    }
    BACKEND_INSTANCE {
        string id PK
        int active_connections
        string status "active|down"
    }
    REQUEST_LOG {
        string id PK
        string tenant_id FK
        string gateway_instance_id FK
        string backend_instance_id FK
        datetime received_at
        int latency_ms
        string result "success|error|timeout"
    }
```
