# Enhance sequence — Generate report job

Đây là **enhance**, flow "Background job tính báo cáo tổng hợp định kỳ". So với base, job không còn chạy 1 câu aggregate chung cho toàn bộ dữ liệu rồi group theo tenant_id — mà lặp qua từng tenant, thiết lập tenant context cho từng connection trước khi query, để row-level security đảm bảo mỗi vòng lặp chỉ thấy đúng dữ liệu của 1 tenant.

```mermaid
sequenceDiagram
    participant Scheduler as Cron Scheduler
    participant Job as ReportJob Worker
    participant TenantDir as Tenant Registry
    participant DB as Shared Database (đã bật RLS)
    participant ReportStore as Report Storage

    Scheduler->>Job: Kích hoạt job tính báo cáo hàng đêm
    Job->>TenantDir: Lấy danh sách tenant_id đang active
    loop với mỗi tenant_id
        Job->>DB: SET app.current_tenant_id = :tenant_id
        Job->>DB: SELECT COUNT(*) FROM tasks
        Note over DB: RLS đảm bảo connection này chỉ thấy dữ liệu của đúng 1 tenant
        DB-->>Job: Kết quả aggregate chỉ của tenant hiện tại
        Job->>ReportStore: Lưu report theo path riêng của tenant
    end
    Job-->>Scheduler: Job hoàn tất
    Note over Job: Test tự động xác nhận báo cáo của tenant A không lẫn dữ liệu tenant B
```
