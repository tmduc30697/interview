# Base ERD — Load balancer checkout trước khi có outlier detection, shedding, pool tách biệt và circuit breaker

Đây là **base**: mô hình dữ liệu suy luận cho load balancer đứng trước checkout service của một sàn e-commerce, ở trạng thái **trước khi** áp các yêu cầu về outlier detection, shed load, tách pool theo loại traffic, chống race condition đếm tải, và circuit breaker phối hợp. Base chỉ cần đủ: cấu hình LB có thể tự chuyển giữa round-robin và least-connection (đúng như mô tả vai trò hiện tại của flow), một pool instance duy nhất phục vụ mọi loại request, và health check nhị phân đơn giản — đủ để các yêu cầu enhance "có nghĩa" khi so sánh.

```mermaid
erDiagram
    LOAD_BALANCER_CONFIG ||--o{ INSTANCE : "điều phối"
    INSTANCE ||--o{ REQUEST : "phục vụ"

    LOAD_BALANCER_CONFIG {
        string id PK
        string current_algorithm "round_robin | least_connection"
        datetime updated_at
    }
    INSTANCE {
        string id PK
        string host
        string health_status "healthy | unhealthy"
        int active_connections
        boolean running_heavy_job
    }
    REQUEST {
        string id PK
        string type "checkout | view_product"
        string instance_id FK
        datetime routed_at
        int response_status
    }
```
