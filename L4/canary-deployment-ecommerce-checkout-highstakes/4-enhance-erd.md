# Enhance ERD — sau khi có canary deployment cho checkout

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `DEPLOYMENT` thêm `role` (stable/canary) và `traffic_percent`; `TRAFFIC_SPLIT` (mới) — chia traffic theo request, tăng dần từ tỷ lệ rất nhỏ, thay vì 100% ngay như base.
- `PAYMENT_METRIC_WINDOW` (mới) — đo riêng theo từng version, gần thời gian thực, gồm cả tỷ lệ thanh toán thất bại chứ không chỉ 5xx.
- `ROLLBACK_POLICY` + `ROLLBACK_EVENT` (mới) — tiêu chí rollback tự động cho ngưỡng nghiêm trọng, không chờ xác nhận thủ công.
- `ORDER` thêm `handled_by_deployment_id` — biết chính xác đơn nào đang dở dang ở canary khi rollback xảy ra.
- `IN_FLIGHT_ORDER_RESOLUTION` (mới) — xử lý nhất quán các đơn dở dang trên canary lúc rollback.

```mermaid
erDiagram
    CHECKOUT_SERVICE ||--o{ DEPLOYMENT : has
    CHECKOUT_SERVICE ||--o{ TRAFFIC_SPLIT : configures
    CHECKOUT_SERVICE ||--|| ROLLBACK_POLICY : "governed by"
    DEPLOYMENT ||--o{ PAYMENT_METRIC_WINDOW : measured
    DEPLOYMENT ||--o{ ROLLBACK_EVENT : "may trigger on"
    DEPLOYMENT ||--o{ ORDER : "handled by"
    ORDER ||--o| PAYMENT_INTENT : creates
    ORDER ||--o| IN_FLIGHT_ORDER_RESOLUTION : "resolved via"
    ROLLBACK_EVENT ||--o{ IN_FLIGHT_ORDER_RESOLUTION : triggers

    CHECKOUT_SERVICE {
        string id PK
        string name "checkout-service"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string role "stable | canary"
        int traffic_percent
        datetime deployed_at
    }
    TRAFFIC_SPLIT {
        string id PK
        string service_id FK
        string canary_deployment_id FK
        string stable_deployment_id FK
        int canary_percent
        string selection_strategy "per_request"
        datetime updated_at
    }
    PAYMENT_METRIC_WINDOW {
        string id PK
        string deployment_id FK
        datetime window_start
        datetime window_end
        decimal payment_failure_rate
        decimal http_5xx_rate
    }
    ROLLBACK_POLICY {
        string id PK
        string service_id FK
        string metric_type "payment_failure_rate | http_5xx_rate"
        string threshold_type "relative_to_baseline | absolute"
        decimal threshold_value
        int window_minutes
        boolean auto_rollback
    }
    ROLLBACK_EVENT {
        string id PK
        string deployment_id FK
        string triggered_by "auto_policy | manual"
        string reason
        datetime triggered_at
    }
    ORDER {
        string id PK
        string status "pending | completed | failed"
        boolean inventory_reserved
        string handled_by_deployment_id FK
        datetime created_at
    }
    PAYMENT_INTENT {
        string id PK
        string order_id FK
        string gateway_status
        datetime created_at
    }
    IN_FLIGHT_ORDER_RESOLUTION {
        string id PK
        string order_id FK
        string rollback_event_id FK
        string resolution "completed_on_canary | compensated_and_retried_on_stable"
        datetime resolved_at
    }
```
