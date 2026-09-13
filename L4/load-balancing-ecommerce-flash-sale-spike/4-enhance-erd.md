# Enhance ERD — sau khi bổ sung outlier detection, load shedding, tách pool, chống race condition và circuit breaker

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có các entity/field mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `INSTANCE` (sửa) — thêm `latency_p95_ms`, `routing_weight`, `consecutive_5xx_count`, `circuit_state`, `connection_count_version` (yêu cầu 1, 4, 5).
- `OUTLIER_SAMPLE` (mới) — mẫu latency đo định kỳ từng instance, dùng phát hiện instance chậm dù health check vẫn pass (yêu cầu 1).
- `POOL_ROUTING_WEIGHT` (mới) — trọng số route riêng theo từng loại traffic (buy/view) cho từng instance, tách biệt cách đối xử giữa request đọc và ghi (yêu cầu 3).
- `LOAD_SHED_DECISION` (mới) — ghi nhận quyết định shed load (trả 503 kèm Retry-After) khi cả cụm đạt ngưỡng tải, thay vì để tất cả cùng timeout (yêu cầu 2).
- `CONNECTION_COUNTER_UPDATE` (mới) — log mọi lần cập nhật `active_connections`, dùng `resulting_version` để đảm bảo cập nhật atomic, tránh race condition (yêu cầu 4).
- `CIRCUIT_BREAKER_EVENT` (mới) — ghi lại mỗi lần instance bị loại khỏi vòng quay do liên tục lỗi 5xx và các lần thử lại half-open (yêu cầu 5).

```mermaid
erDiagram
    LOAD_BALANCER_CONFIG ||--o{ INSTANCE : "điều phối"
    INSTANCE ||--o{ REQUEST : "phục vụ"
    INSTANCE ||--o{ OUTLIER_SAMPLE : "được đo latency"
    INSTANCE ||--o{ POOL_ROUTING_WEIGHT : "có trọng số theo loại traffic"
    REQUEST ||--o| LOAD_SHED_DECISION : "có thể bị shed"
    INSTANCE ||--o{ CONNECTION_COUNTER_UPDATE : "nhận cập nhật atomic"
    INSTANCE ||--o{ CIRCUIT_BREAKER_EVENT : "có lịch sử chuyển trạng thái"

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
        int latency_p95_ms
        decimal routing_weight
        int consecutive_5xx_count
        string circuit_state "closed | open | half_open"
        int connection_count_version
    }
    REQUEST {
        string id PK
        string type "checkout | view_product"
        string instance_id FK
        datetime routed_at
        int response_status
    }
    OUTLIER_SAMPLE {
        string id PK
        string instance_id FK
        int latency_ms
        datetime sampled_at
    }
    POOL_ROUTING_WEIGHT {
        string instance_id PK, FK
        string pool_type PK "checkout_write | view_product_read"
        decimal weight
    }
    LOAD_SHED_DECISION {
        string id PK
        string request_id FK
        string decision "shed | serve"
        int retry_after_seconds
        decimal cluster_load_snapshot
        datetime decided_at
    }
    CONNECTION_COUNTER_UPDATE {
        string id PK
        string instance_id FK
        int delta
        int resulting_version
        string source_request_id FK
        datetime recorded_at
    }
    CIRCUIT_BREAKER_EVENT {
        string id PK
        string instance_id FK
        string from_state "closed | open | half_open"
        string to_state "closed | open | half_open"
        string reason
        datetime occurred_at
    }
```
