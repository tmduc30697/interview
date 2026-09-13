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
