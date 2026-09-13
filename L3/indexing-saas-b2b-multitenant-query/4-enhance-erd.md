# ERD — Enhance (sau khi chuẩn hoá index đa tenant)

Đây là **enhance**: ERD base cộng thêm các entity/cột phục vụ đúng 5 yêu cầu của đề bài. So với base, `TABLE_INDEX` được bổ sung cột `jsonb_path` và quy ước `leads_with_tenant_id` nay luôn là true cho mọi composite index (yêu cầu 1); `TENANT_TABLE_STATS` là entity mới theo dõi thống kê ANALYZE theo từng tenant để xử lý độ lệch cardinality (yêu cầu 2); `RECORD` có thêm cột `custom_fields JSONB` cùng entity `CUSTOM_FIELD_DEFINITION` cho phép tenant tự khai báo field động (yêu cầu 3); `REPORTING_REPLICA` tách index/luồng report nặng khỏi luồng giao dịch (yêu cầu 4); `INDEX_MIGRATION_JOB` ghi nhận việc build index luôn chạy online, không khoá ghi (yêu cầu 5).

```mermaid
erDiagram
    TENANT ||--o{ TENANT_USER : employs
    TENANT ||--o{ RECORD : owns
    TENANT_USER ||--o{ RECORD : "creates/updates"
    TENANT ||--o{ CUSTOM_FIELD_DEFINITION : defines
    TENANT ||--o{ TENANT_TABLE_STATS : "tracked for"

    RECORD ||--o{ TABLE_INDEX : "has index defined on (catalog-level, not per-row)"
    RECORD ||--o{ TENANT_TABLE_STATS : "stats collected on (catalog-level, not per-row)"

    REPORTING_REPLICA ||--o{ TABLE_INDEX : "may host separate reporting-only index copy"
    INDEX_MIGRATION_JOB ||--|| TABLE_INDEX : creates

    TENANT {
        string tenant_id PK
        string company_name
        string plan_tier
        datetime created_at
    }

    TENANT_USER {
        string user_id PK
        string tenant_id FK
        string email
        string role
        datetime created_at
    }

    RECORD {
        string record_id PK
        string tenant_id FK
        string owner_user_id FK
        string title
        string status
        decimal amount
        jsonb custom_fields
        datetime created_at
        datetime updated_at
    }

    CUSTOM_FIELD_DEFINITION {
        string field_def_id PK
        string tenant_id FK
        string field_key
        string field_type
        datetime created_at
    }

    TABLE_INDEX {
        string index_id PK
        string table_name
        string index_name
        string column_order
        string index_type
        boolean leads_with_tenant_id
        string jsonb_path
    }

    TENANT_TABLE_STATS {
        string stats_id PK
        string tenant_id FK
        string table_name
        bigint row_estimate
        datetime last_analyzed_at
        string planner_flag
    }

    REPORTING_REPLICA {
        string replica_id PK
        string role
        int replica_lag_seconds
        datetime connected_since
    }

    INDEX_MIGRATION_JOB {
        string job_id PK
        string index_id FK
        string method
        string status
        datetime started_at
        datetime completed_at
        boolean write_blocking_detected
    }
```
