# Enhance ERD — sau khi có blue-green deployment

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `DEPLOYMENT` thêm `color` (blue/green) và `status` mở rộng (staging/smoke_testing/live/retiring/rolled_back) — thay vì chỉ có đúng 1 deployment "active".
- `SMOKE_TEST_RUN` (mới) — bắt buộc pass trước khi router chuyển traffic thật.
- `ROUTER_SWITCH_EVENT` (mới) — thao tác chuyển traffic gần như tức thời và có thể đảo ngược (`reversible_until`), thay cho việc router chỉ đơn giản trỏ theo instance đang chạy như base.
- `DB_MIGRATION` thêm `backward_compatible` + `phase` (expand/contract) — đảm bảo blue vẫn chạy đúng trong giai đoạn chuyển tiếp.
- `ROLLBACK_POLICY` + `ROLLBACK_EVENT` (mới) — tiêu chí rollback tự động, không chờ phát hiện thủ công.
- `CONNECTION_DRAIN` (mới) — xử lý có kiểm soát các connection đang mở trên blue khi traffic đã chuyển sang green.

```mermaid
erDiagram
    SERVICE ||--o{ DEPLOYMENT : has
    SERVICE ||--o{ ROUTER_SWITCH_EVENT : records
    SERVICE ||--o{ DB_MIGRATION : applies
    SERVICE ||--|| ROLLBACK_POLICY : "governed by"
    DEPLOYMENT ||--o{ SMOKE_TEST_RUN : "tested by"
    DEPLOYMENT ||--o{ CLIENT_CONNECTION : serves
    DEPLOYMENT ||--o| CONNECTION_DRAIN : "drains via"
    ROUTER_SWITCH_EVENT ||--o| ROLLBACK_EVENT : "may trigger"

    SERVICE {
        string id PK
        string name "core-api"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string color "blue | green"
        string status "staging | smoke_testing | live | retiring | rolled_back"
        datetime deployed_at
    }
    SMOKE_TEST_RUN {
        string id PK
        string deployment_id FK
        string test_name
        string result "pass | fail"
        datetime run_at
    }
    ROUTER_SWITCH_EVENT {
        string id PK
        string service_id FK
        string from_deployment_id FK
        string to_deployment_id FK
        datetime switched_at
        datetime reversible_until
        string status "active | reverted"
    }
    DB_MIGRATION {
        string id PK
        string service_id FK
        string version
        string description
        boolean backward_compatible
        string phase "expand | contract"
        datetime applied_at
    }
    ROLLBACK_POLICY {
        string id PK
        string service_id FK
        decimal error_rate_threshold
        int window_minutes
        string metric_source
    }
    ROLLBACK_EVENT {
        string id PK
        string router_switch_event_id FK
        string triggered_by "auto_policy | manual"
        string reason
        string reverted_to_deployment_id FK
        datetime triggered_at
    }
    CLIENT_CONNECTION {
        string id PK
        string deployment_id FK
        string type "websocket | long_request"
        datetime opened_at
        string status
    }
    CONNECTION_DRAIN {
        string id PK
        string deployment_id FK
        string connection_type
        int count_at_switch
        string drain_status
        datetime completed_at
    }
```
