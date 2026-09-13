# Enhance ERD — Circuit breaker độc lập theo carrier + fallback an toàn

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `CARRIER_BREAKER_STATE` (mới) — 1 breaker riêng cho mỗi carrier, ngưỡng lỗi/thời gian mở cấu hình độc lập theo từng carrier, không dùng chung 1 breaker toàn cục (yêu cầu 1).
- `FALLBACK_PRIORITY` (mới) — thứ tự ưu tiên carrier dự phòng theo vùng phục vụ, chi phí, hoặc SLA hiện tại, dùng khi breaker carrier chính mở (yêu cầu 2).
- `RETRY_POLICY` (mới) — phân loại lỗi nào được retry, có backoff_base_ms và jitter_max_ms để tránh dồn request vào carrier lúc cao điểm (yêu cầu 3, 4).
- `SHIPMENT_CALL_LOG` (sửa) — thêm `error_type` và `verified_not_created` để ghi rõ đã xác minh qua API tra cứu trước khi retry hay chưa (yêu cầu 3).
- `CARRIER_METRIC` (mới) — đo tỉ lệ lỗi, thời gian breaker mở trong ngày, tỉ lệ đơn phải fallback, và chi phí phát sinh do fallback, theo từng carrier (yêu cầu 5).

```mermaid
erDiagram
    ORDER ||--|| SHIPMENT : "generates"
    CARRIER ||--o{ SHIPMENT : "fulfills"
    SHIPMENT ||--o{ SHIPMENT_CALL_LOG : "logs calls"
    SHIPMENT_CALL_LOG }o--|| RETRY_POLICY : "governed by"
    CARRIER ||--|| CARRIER_BREAKER_STATE : "has own breaker"
    CARRIER ||--o{ FALLBACK_PRIORITY : "ranked in"
    CARRIER ||--o{ CARRIER_METRIC : "measured per"

    ORDER {
        string id PK
        string status "pending | shipped | failed"
    }
    CARRIER {
        string id PK
        string name
        string region
        decimal base_cost
    }
    SHIPMENT {
        string id PK
        string order_id FK
        string carrier_id FK
        string tracking_code
        string status "pending | created | failed"
    }
    SHIPMENT_CALL_LOG {
        string id PK
        string shipment_id FK
        string call_type "create | track"
        string status "success | failed | timeout"
        string error_type "timeout_unconfirmed | confirmed_fail | 5xx"
        boolean verified_not_created
        int attempt_number
        datetime called_at
    }
    RETRY_POLICY {
        string id PK
        string error_type "timeout_unconfirmed | 5xx | confirmed_fail"
        boolean retryable
        int backoff_base_ms
        int jitter_max_ms
    }
    CARRIER_BREAKER_STATE {
        string id PK
        string carrier_id FK
        string state "closed | open | half_open"
        decimal error_rate_threshold
        int error_rate_window_seconds
        datetime opened_at
    }
    FALLBACK_PRIORITY {
        string id PK
        string region
        string carrier_id FK
        int priority_order
        string basis "region | cost | current_sla"
    }
    CARRIER_METRIC {
        string id PK
        string carrier_id FK
        date window_date
        decimal error_rate
        int breaker_open_duration_ms
        decimal fallback_rate
        decimal extra_cost
    }
```
