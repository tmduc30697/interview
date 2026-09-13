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
