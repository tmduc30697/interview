# Enhance ERD — sau khi canary nhận diện đa phiên bản app

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `MIN_SUPPORTED_APP_VERSION` (mới) — xác định rõ tập phiên bản app tối thiểu cần hỗ trợ dựa trên số liệu user thực tế, dùng làm căn cứ test hợp đồng thay vì chỉ test app mới nhất.
- `CANARY_TRAFFIC_STRATIFICATION` (mới) — đảm bảo traffic canary có đủ tỷ lệ app cũ lẫn mới, không dồn ngẫu nhiên như base.
- `CLIENT_VERSION_ERROR_METRIC` (mới) — tách error/parse-error rate theo từng phiên bản app, thay vì gộp chung.
- `SCHEMA_COMPATIBILITY_PLAN` + `ROLLBACK_EVENT` (mới) — rollback có kế hoạch đảm bảo dữ liệu ghi theo schema mới vẫn đọc được bởi code cũ.

```mermaid
erDiagram
    BACKEND_DEPLOYMENT ||--o{ MIN_SUPPORTED_APP_VERSION : "tested against"
    BACKEND_DEPLOYMENT ||--o{ CANARY_TRAFFIC_STRATIFICATION : configures
    BACKEND_DEPLOYMENT ||--o{ CLIENT_VERSION_ERROR_METRIC : measured
    BACKEND_DEPLOYMENT ||--o| SCHEMA_COMPATIBILITY_PLAN : "planned with"
    BACKEND_DEPLOYMENT ||--o{ ROLLBACK_EVENT : "may trigger on"

    APP_CLIENT_VERSION {
        string id PK
        string version
        int active_user_count
    }
    BACKEND_DEPLOYMENT {
        string id PK
        string version
        string role "stable | canary"
        int traffic_percent
    }
    MIN_SUPPORTED_APP_VERSION {
        string id PK
        string backend_version FK
        string app_version
        int active_user_count_at_test_time
        string contract_test_result
    }
    CANARY_TRAFFIC_STRATIFICATION {
        string id PK
        string backend_deployment_id FK
        string app_version
        int target_percent_of_canary
    }
    CLIENT_VERSION_ERROR_METRIC {
        string id PK
        string backend_version FK
        string app_client_version
        datetime window_start
        datetime window_end
        decimal error_rate
        decimal parse_error_rate
    }
    SCHEMA_COMPATIBILITY_PLAN {
        string id PK
        string backend_version FK
        string new_fields_written
        string backward_read_strategy "ignorable_by_old_version | requires_shim"
    }
    ROLLBACK_EVENT {
        string id PK
        string backend_deployment_id FK
        string triggered_by "client_version_error_metric | manual"
        string affected_app_versions
        string reason
        datetime triggered_at
    }
```
