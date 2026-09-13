# Enhance ERD — Thêm tín hiệu drain, theo dõi connection, circuit breaker động, rolling restart run

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi:
- `INSTANCE` có thêm trạng thái `draining` và cột `schema_version` — đáp ứng yêu cầu 1 (tín hiệu shutdown chủ động) và yêu cầu 3 (nhận biết version khác nhau giữa các instance khi rolling restart đang chạy dở).
- Entity mới `DRAIN_SIGNAL` — sự kiện instance chủ động báo "sắp shutdown" gửi tới gateway trước khi ngừng nhận connection, độc lập với health check polling — đáp ứng yêu cầu 1.
- Entity mới `KEEP_ALIVE_CONNECTION` — theo dõi từng kết nối keep-alive đang mở gắn với instance nào, để gateway biết kết nối nào cần bị đóng/không tái sử dụng khi instance đó drain — đáp ứng yêu cầu 2.
- Entity mới `CIRCUIT_BREAKER` gắn theo service, có `error_budget_multiplier` được nới rộng tạm thời trong lúc rolling restart — đáp ứng yêu cầu 4.
- Entity mới `ROLLING_RESTART_RUN` — ghi nhận 1 lần rolling restart của 1 service (đang restart instance nào, đã xong bao nhiêu, số request/lỗi quan sát được) — làm nền cho bài test toàn trình đo tỷ lệ lỗi — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    SERVICE ||--o{ INSTANCE : "has"
    SERVICE ||--o| CIRCUIT_BREAKER : "protected by"
    SERVICE ||--o{ ROLLING_RESTART_RUN : "restarted via"
    INSTANCE ||--o{ DRAIN_SIGNAL : emits
    INSTANCE ||--o{ KEEP_ALIVE_CONNECTION : serves
    ROLLING_RESTART_RUN ||--o{ DRAIN_SIGNAL : includes

    SERVICE {
        string id PK
        string name
    }
    INSTANCE {
        string id PK
        string service_id FK
        string host
        int port
        string status "healthy | draining | unhealthy"
        string schema_version
        datetime last_health_check_at
    }
    DRAIN_SIGNAL {
        string id PK
        string instance_id FK
        string reason "sigterm | rolling_restart"
        datetime signaled_at
        datetime deregistered_at
    }
    KEEP_ALIVE_CONNECTION {
        string id PK
        string instance_id FK
        string client_id
        string status "open | closing | closed"
        datetime established_at
    }
    CIRCUIT_BREAKER {
        string id PK
        string service_id FK
        string state "closed | open | half_open"
        float error_threshold
        float error_budget_multiplier "1.0 mac dinh, tang tam thoi khi rolling restart"
        datetime widened_until
    }
    ROLLING_RESTART_RUN {
        string id PK
        string service_id FK
        int total_instances
        int instances_restarted
        int request_count
        int error_count
        float sla_error_threshold
        string status "running | completed | failed"
    }
```
