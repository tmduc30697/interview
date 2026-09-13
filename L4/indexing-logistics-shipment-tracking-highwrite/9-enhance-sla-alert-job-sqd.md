# Enhance - Flow job định kỳ tìm vận đơn trễ SLA

Đây là **enhance**, sequence diagram cho flow mới **sla-alert-job**: job chạy định kỳ mỗi phút để tìm các vận đơn đã quá estimated_delivery nhưng chưa delivered. Flow này hoàn toàn mới so với base (base chưa có index nào phục vụ dạng truy vấn này), được thêm để minh hoạ index riêng cho cảnh báo SLA giải quyết yêu cầu thứ tư của đề bài.

```mermaid
sequenceDiagram
    participant Scheduler as Cron mỗi phút
    participant Worker as SLA Alert Worker
    participant DB as PostgreSQL
    participant Notifier as Hệ thống cảnh báo

    Scheduler->>Worker: Kích hoạt job kiểm tra SLA
    Worker->>DB: SELECT tracking_code, estimated_delivery, status FROM shipments WHERE status khac delivered AND estimated_delivery < now()
    Note over DB: planner dùng index riêng idx_sla_alert, partial WHERE status khac delivered kết hợp composite trên estimated_delivery, chỉ quét tập con đơn chưa giao thay vì full table scan hàng triệu dòng mỗi phút
    DB-->>Worker: danh sách vận đơn trễ SLA
    Worker->>Notifier: Gửi cảnh báo cho từng vận đơn trễ
    Notifier-->>Worker: đã gửi
    Worker-->>Scheduler: Hoàn tất job
```
