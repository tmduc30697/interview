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
