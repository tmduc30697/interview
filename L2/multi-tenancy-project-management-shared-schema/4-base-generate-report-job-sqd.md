# Base sequence — Generate report job

Đây là **base**, flow "Background job tính báo cáo tổng hợp định kỳ" — đại diện cho nhóm thao tác nền (background job/cron) mà đề bài yêu cầu phải tôn trọng ranh giới tenant. Chọn flow này vì đây đúng là ví dụ đề bài nêu: 1 job tính báo cáo có nguy cơ gộp nhầm dữ liệu nhiều tenant vào 1 kết quả chung nếu câu query aggregate viết thiếu group/filter theo tenant_id.

```mermaid
sequenceDiagram
    participant Scheduler as Cron Scheduler
    participant Job as ReportJob Worker
    participant DB as Shared Database
    participant ReportStore as Report Storage

    Scheduler->>Job: Kích hoạt job tính báo cáo hàng đêm
    Job->>DB: SELECT tenant_id, COUNT(*) FROM tasks GROUP BY tenant_id
    Note over Job,DB: Câu query aggregate viết thủ công, phụ thuộc vào người viết job nhớ group/filter theo tenant_id
    DB-->>Job: Kết quả aggregate theo từng tenant
    Job->>ReportStore: Lưu report theo tenant_id
    Job-->>Scheduler: Job hoàn tất
    Note over Job,DB: Không có test tự động nào xác nhận job không gộp nhầm dữ liệu giữa các tenant
```
