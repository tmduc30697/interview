# Enhance ERD — sau khi có rate limit per-API-key, phân tán, theo gói

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `PLAN_LIMIT` (mới) — giới hạn request/giây và request/tháng cụ thể theo từng gói.
- `RATE_LIMIT_BUCKET` (mới) — trạng thái token bucket theo từng API key, lưu tập trung (Redis) để chính xác dù nhiều instance gateway chạy song song.
- `MONTHLY_USAGE_COUNTER` (mới) — đếm riêng theo tháng, độc lập với bucket giây, phục vụ cả giới hạn tháng lẫn dashboard usage.
- `COORDINATOR_HEALTH_STATE` (mới) — trạng thái Redis/coordinator và chiến lược fail-open/fail-closed khi nó chậm/down.

```mermaid
erDiagram
    CUSTOMER ||--o{ API_KEY : owns
    API_KEY ||--o{ API_REQUEST_LOG : "makes calls logged as"
    CUSTOMER }o--|| PLAN_LIMIT : "plan matches"
    API_KEY ||--|| RATE_LIMIT_BUCKET : "throttled per second via"
    API_KEY ||--o{ MONTHLY_USAGE_COUNTER : "tracked monthly by"
    RATE_LIMIT_BUCKET }o--|| COORDINATOR_HEALTH_STATE : "checked against"

    CUSTOMER {
        string id PK
        string name
        string plan "free | pro | enterprise"
        datetime created_at
    }
    API_KEY {
        string id PK
        string customer_id FK
        string key_value
        string status "active | revoked"
        datetime created_at
    }
    API_REQUEST_LOG {
        string id PK
        string api_key_id FK
        string endpoint
        int response_status
        datetime requested_at
    }
    PLAN_LIMIT {
        string id PK
        string plan "free | pro | enterprise"
        int requests_per_second
        int requests_per_month
    }
    RATE_LIMIT_BUCKET {
        string id PK
        string api_key_id FK
        string algorithm "token_bucket"
        decimal tokens_remaining
        int bucket_capacity
        datetime last_refill_at
    }
    MONTHLY_USAGE_COUNTER {
        string id PK
        string api_key_id FK
        string month "YYYY-MM"
        int request_count
    }
    COORDINATOR_HEALTH_STATE {
        string id PK
        string coordinator_name "redis_rate_limit_cluster"
        string status "healthy | degraded | down"
        string fallback_strategy "fail_open | fail_closed"
        datetime updated_at
    }
```
