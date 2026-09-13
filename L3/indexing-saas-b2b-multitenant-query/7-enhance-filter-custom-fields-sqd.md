# Sequence Diagram — Enhance: Filter Custom Fields

Đây là **enhance**, flow hoàn toàn mới chưa tồn tại ở base: tenant lọc dữ liệu theo custom field lưu trong cột `custom_fields JSONB` dùng chung. Flow này được chọn vì thể hiện trực tiếp yêu cầu 3 của đề bài — B-tree không hiệu quả cho field động này, nên cần GIN index (kết hợp `tenant_id` để không quét chéo dữ liệu tenant khác trong cùng cột JSONB dùng chung).

```mermaid
sequenceDiagram
    actor TenantUser as Tenant User
    participant API as App Service
    participant DB as Postgres (shared RECORD table)
    participant Planner as Query Planner

    TenantUser->>API: GET /records?custom.priority=urgent
    API->>DB: SELECT * FROM record WHERE tenant_id = 'A' AND custom_fields @> '{"priority": "urgent"}'
    DB->>Planner: Choose execution plan
    Note over Planner: GIN index idx_record_tenant_custom_fields on (tenant_id, custom_fields) is available
    Planner-->>DB: Use GIN index for jsonb containment, still scoped to Tenant A's slice first
    DB-->>API: Matching rows returned without scanning other tenants' custom_fields blobs
    API-->>TenantUser: Filtered records list, fast even though the field is dynamic per tenant

    Note over API,DB: This flow did not exist at base, custom_fields column, CUSTOM_FIELD_DEFINITION and its GIN index are all new from enhance
```
