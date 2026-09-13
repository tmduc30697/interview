# Sequence Diagram — Base: Tenant Query Records

Đây là **base**, flow "tenant truy vấn danh sách dữ liệu của mình" trên bảng `RECORD` dùng chung. Flow này được chọn vì nó chính là nạn nhân trực tiếp của 2 vấn đề đầu tiên trong đề bài: composite index không đặt `tenant_id` đầu tiên (một tính năng cũ tạo `idx_record_status_tenant (status, tenant_id)` sai thứ tự) và độ lệch dữ liệu lớn giữa các tenant khiến planner chọn plan tệ. Diagram cho thấy cả tenant lớn lẫn tenant nhỏ đều bị ảnh hưởng, kể cả khi chỉ một bên có nhiều dữ liệu.

```mermaid
sequenceDiagram
    actor TenantUserA as Tenant A User (large tenant)
    actor TenantUserB as Tenant B User (small tenant)
    participant API as App Service
    participant DB as Postgres (shared RECORD table)
    participant Planner as Query Planner

    TenantUserA->>API: GET /records?status=open
    API->>DB: SELECT * FROM record WHERE tenant_id = 'A' AND status = 'open' ORDER BY created_at DESC LIMIT 50
    DB->>Planner: Choose execution plan
    Note over Planner: Only composite index available is idx_record_status_tenant (status, tenant_id), not (tenant_id, status)
    Note over Planner: Table stats are stale, last ANALYZE ran days ago and does not reflect current tenant sizes
    Planner-->>DB: Plan chosen, scan by status first across ALL tenants, then filter tenant_id
    DB-->>API: Rows returned after scanning many irrelevant rows from other tenants
    API-->>TenantUserA: Slow response, records list

    TenantUserB->>API: GET /records?status=open
    API->>DB: SELECT * FROM record WHERE tenant_id = 'B' AND status = 'open' ORDER BY created_at DESC LIMIT 50
    DB->>Planner: Choose execution plan
    Note over Planner: Same shared, wrongly-ordered index, Tenant B rows are a tiny fraction of the status=open set
    Planner-->>DB: Plan chosen, same status-first scan touches millions of Tenant A rows to find few Tenant B rows
    DB-->>API: Slow response despite Tenant B owning very little data
    API-->>TenantUserB: Slow records list, noisy-neighbor impact from Tenant A's volume

    Note over API,DB: No isolation between tenants, both pay the cost of one shared and wrongly-ordered composite index
```
