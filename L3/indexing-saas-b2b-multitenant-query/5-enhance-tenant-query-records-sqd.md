# Sequence Diagram — Enhance: Tenant Query Records

Đây là **enhance** của flow đã có ở base ("tenant truy vấn danh sách dữ liệu của mình"). So với base, flow này thay đổi ở 2 điểm: composite index nay là `idx_record_tenant_status (tenant_id, status)` — `tenant_id` luôn đứng đầu (yêu cầu 1) — và thống kê `TENANT_TABLE_STATS` được refresh định kỳ theo từng tenant để planner không còn bị đánh lừa bởi độ lệch cardinality (yêu cầu 2). Kết quả là cả tenant lớn lẫn tenant nhỏ đều chỉ chạm vào đúng dữ liệu của mình.

```mermaid
sequenceDiagram
    actor TenantUserA as Tenant A User (large tenant)
    actor TenantUserB as Tenant B User (small tenant)
    participant API as App Service
    participant DB as Postgres (shared RECORD table)
    participant Planner as Query Planner
    participant Stats as TENANT_TABLE_STATS Monitor

    Stats->>DB: Scheduled ANALYZE, refresh row_estimate per tenant_id on RECORD
    Note over Stats,DB: Per-tenant stats catch skew early instead of relying on one stale table-wide estimate

    TenantUserA->>API: GET /records?status=open
    API->>DB: SELECT * FROM record WHERE tenant_id = 'A' AND status = 'open' ORDER BY created_at DESC LIMIT 50
    DB->>Planner: Choose execution plan
    Note over Planner: idx_record_tenant_status (tenant_id, status) leads with tenant_id
    Planner-->>DB: Index scan narrows to Tenant A's own slice of the index first, then filters status
    DB-->>API: Rows returned quickly, only Tenant A's own data touched
    API-->>TenantUserA: Fast records list

    TenantUserB->>API: GET /records?status=open
    API->>DB: SELECT * FROM record WHERE tenant_id = 'B' AND status = 'open' ORDER BY created_at DESC LIMIT 50
    DB->>Planner: Choose execution plan
    Note over Planner: Same tenant_id-first index, fresh per-tenant stats show Tenant B's small row_estimate
    Planner-->>DB: Index scan touches only Tenant B's tiny data set
    DB-->>API: Fast response, unaffected by Tenant A's volume
    API-->>TenantUserB: Fast records list, isolation preserved

    Note over API,DB: tenant_id-first composite index plus fresh per-tenant stats remove both the full-scan risk and the noisy-neighbor effect seen in base
```
