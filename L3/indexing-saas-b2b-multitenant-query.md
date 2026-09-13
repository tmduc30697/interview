# SaaS B2B tối ưu index cho truy vấn đa tenant dùng chung bảng

**Hệ thống:** SaaS B2B multi-tenant dùng shared schema — nhiều công ty khách hàng (tenant) lưu dữ liệu chung trong cùng một bộ bảng, phân biệt bằng cột `tenant_id`, với quy mô dữ liệu lệch rất lớn giữa các tenant (tenant lớn hàng triệu dòng, tenant nhỏ chỉ vài trăm dòng).

**Vai trò của flow:** Đảm bảo mỗi tenant tra cứu dữ liệu của riêng mình luôn nhanh và cách ly đúng, bất kể tenant khác đang có khối lượng dữ liệu hoặc truy vấn lớn tới đâu (tránh noisy-neighbor ảnh hưởng lẫn nhau qua tầng index/cache dùng chung).

**Yêu cầu cụ thể:**
- Mọi composite index trên các bảng dùng chung đều phải đặt `tenant_id` làm cột đầu tiên để mỗi truy vấn luôn lọc được theo tenant trước — nếu một tính năng mới quên tuân thủ quy tắc này, query sẽ rơi vào full table scan chậm dần và có nguy cơ vô tình đọc chéo dữ liệu tenant khác do lỗi logic đi kèm.
- Do độ lệch dữ liệu lớn giữa các tenant, query planner có thể chọn plan khác nhau cho cùng một câu query tùy vào cardinality của tenant đang chạy (tenant nhỏ có thể bị ép dùng index kém hiệu quả, tenant lớn có thể bị planner chọn nhầm sang full scan) — cần theo dõi thống kê (ANALYZE) và xử lý rõ trường hợp planner chọn sai plan.
- Một số tenant cần lưu field tùy biến riêng (custom fields) qua cột JSONB dùng chung cho mọi tenant — B-tree thường không index hiệu quả cho các field động này, cần đánh giá dùng GIN index hoặc expression index cho từng loại field hay truy vấn phổ biến.
- Khi một tenant lớn chạy report/aggregation nặng, nó có thể chiếm phần lớn buffer pool/index cache dùng chung, khiến các tenant nhỏ khác bị chậm theo dù không liên quan tới truy vấn đó — cần cân nhắc tách index/replica riêng cho luồng reporting nặng khỏi luồng giao dịch thời gian thực.
- Thêm index mới cho một tính năng phải chạy migration online (kiểu CREATE INDEX CONCURRENTLY) trên bảng chứa dữ liệu của toàn bộ tenant cùng lúc — cần đảm bảo quá trình build index không khoá ghi và không làm gián đoạn tenant nào, kể cả tenant đang có traffic cao nhất tại thời điểm migrate.

---

# ERD — Base (trước khi chuẩn hoá index đa tenant)

Đây là **base**: mô hình dữ liệu suy luận cho hệ thống SaaS B2B multi-tenant dùng shared schema *trước khi* áp các quy tắc tối ưu index của đề bài. Đề bài giả định đã tồn tại `TENANT`, `TENANT_USER` và một bảng dữ liệu nghiệp vụ dùng chung `RECORD` (đại diện cho loại bảng "shared table" bất kỳ — invoice, lead, ticket... — có cột `tenant_id`, quy mô lệch lớn giữa các tenant) — nếu không có các entity này thì việc "tối ưu index cho truy vấn đa tenant" sẽ không có nghĩa. ERD còn đưa `TABLE_INDEX` vào như metadata catalog (kiểu `pg_indexes`) để thể hiện đúng vấn đề gốc: ở base, thứ tự cột trong composite index KHÔNG được chuẩn hoá nhất quán — cột `leads_with_tenant_id` có giá trị hỗn hợp tuỳ theo index được tạo bởi tính năng nào, dẫn tới rủi ro full table scan nêu trong đề bài.

```mermaid
erDiagram
    TENANT ||--o{ TENANT_USER : employs
    TENANT ||--o{ RECORD : owns
    TENANT_USER ||--o{ RECORD : "creates/updates"
    RECORD ||--o{ TABLE_INDEX : "has index defined on (catalog-level, not per-row)"

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
        datetime created_at
        datetime updated_at
    }

    TABLE_INDEX {
        string index_id PK
        string table_name
        string index_name
        string column_order
        string index_type
        boolean leads_with_tenant_id
    }
```

---

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

---

# Sequence Diagram — Base: Add Index Migration

Đây là **base**, flow "thêm index mới cho một tính năng" theo cách làm cũ. Flow này được chọn để làm tiền đề đối chiếu trực tiếp với yêu cầu cuối của đề bài (migration online, không khoá ghi) — ở base, chưa có quy ước bắt buộc dùng `CREATE INDEX CONCURRENTLY` lẫn quy ước bắt buộc `tenant_id` phải là cột đầu, nên migration này vừa khoá bảng vừa vô tình tạo ra chính cái index sai thứ tự đã gây chậm ở flow truy vấn.

```mermaid
sequenceDiagram
    actor Dev as Feature Developer
    participant Migration as Migration Tool
    participant DB as Postgres (shared RECORD table)
    actor TenantUserA as Tenant A User (highest traffic)

    Dev->>Migration: Write migration, CREATE INDEX idx_record_status_tenant ON record(status, tenant_id)
    Note over Dev,Migration: No convention enforced yet, column order left to whoever writes the feature
    Migration->>DB: Run plain CREATE INDEX during deploy window

    DB->>DB: Acquire lock that blocks concurrent writes while building the index
    Note over DB: Build must scan and sort the entire RECORD table, dominated by Tenant A's millions of rows

    TenantUserA->>DB: Attempt to insert/update a record during the build
    DB-->>TenantUserA: Write blocked/timed out until index build finishes

    DB-->>Migration: Index build completed
    Migration-->>Dev: Migration finished, but Tenant A saw write timeouts during deploy

    Note over DB: The resulting idx_record_status_tenant (status, tenant_id) is also the wrongly-ordered index later causing slow queries
```

---

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

---

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

---

# Sequence Diagram — Enhance: Add Index Migration

Đây là **enhance** của flow đã có ở base ("thêm index mới cho một tính năng"). So với base, flow này thay đổi ở 2 điểm: có bước review bắt buộc kiểm tra `tenant_id` là cột đầu tiên trước khi cho migrate (yêu cầu 1), và migration luôn chạy bằng `CREATE INDEX CONCURRENTLY`, được `INDEX_MIGRATION_JOB` theo dõi để đảm bảo không khoá ghi kể cả với tenant có traffic cao nhất (yêu cầu 5) — trái ngược hoàn toàn với base, nơi migration dùng `CREATE INDEX` thường và khoá bảng suốt quá trình build.

```mermaid
sequenceDiagram
    actor Dev as Feature Developer
    participant Review as Index Convention Check
    participant Migration as Migration Tool
    participant DB as Postgres (shared RECORD table)
    actor TenantUserA as Tenant A User (highest traffic)

    Dev->>Review: Submit migration, CREATE INDEX ON record(tenant_id, new_feature_flag)
    Review-->>Dev: Approved, tenant_id is the first column as required

    Dev->>Migration: Run migration in production
    Migration->>DB: CREATE INDEX CONCURRENTLY idx_record_tenant_new_feature ON record(tenant_id, new_feature_flag)
    Migration->>Migration: Create INDEX_MIGRATION_JOB record, method=CONCURRENTLY, status=running
    Note over DB: CONCURRENTLY builds the index in the background without an ACCESS EXCLUSIVE lock, writes stay unblocked

    TenantUserA->>DB: Continue inserting/updating records normally during the build
    DB-->>TenantUserA: Writes succeed with no timeout

    DB->>DB: Build index in multiple passes, validate for conflicting rows
    DB-->>Migration: Index build completed and marked valid
    Migration->>Migration: Update INDEX_MIGRATION_JOB, status=completed, write_blocking_detected=false
    Migration-->>Dev: Migration succeeded, no downtime, no blocked tenant
```

---

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

---

# Sequence Diagram — Enhance: Run Heavy Report

Đây là **enhance**, flow hoàn toàn mới chưa tồn tại ở base: một tenant lớn chạy report/aggregation nặng trên `REPORTING_REPLICA` thay vì trên bảng transactional chính. Flow này được chọn vì thể hiện trực tiếp yêu cầu 4 của đề bài — tách index/replica riêng cho luồng reporting nặng, để tenant nhỏ khác không bị chậm theo do buffer pool/index cache dùng chung bị tenant lớn chiếm dụng.

```mermaid
sequenceDiagram
    actor TenantAdmin as Tenant A Admin (large tenant)
    participant API as Reporting Service
    participant Router as Query Router
    participant Primary as Postgres Primary (transactional)
    participant Replica as Reporting Replica

    TenantAdmin->>API: Request heavy aggregation report, monthly totals across millions of records
    API->>Router: Route report query
    Note over Router: Reporting-style queries are routed to Replica, not Primary
    Router->>Replica: Run aggregation using replica-only reporting indexes
    Replica-->>Router: Aggregated results, buffer pool usage isolated to the replica
    Router-->>API: Report data
    API-->>TenantAdmin: Report delivered

    Note over Primary: Meanwhile Primary keeps serving normal tenant_id-first indexed queries for every tenant
    Primary->>Primary: Continue serving real-time transactional queries, buffer pool undisturbed by the report

    Note over API,Replica: This isolation path did not exist at base, heavy reports used to run directly on Primary and could evict other tenants' hot data from the shared buffer pool/index cache
```
