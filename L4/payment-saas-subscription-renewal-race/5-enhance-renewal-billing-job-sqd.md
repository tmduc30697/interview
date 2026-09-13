# Enhance sequence — Job renewal với version lock, dừng ngay nếu đã bị hủy

Đây là **enhance**, flow "Job billing tự động renewal" sau khi có version check trước khi charge. So với base, flow này thay đổi ở chỗ: job ghi nhận `version` tại thời điểm đọc subscription, và ngay trước khi gọi cổng thanh toán, job kiểm tra lại version/status 1 lần nữa — nếu subscription đã bị hủy (version thay đổi) thì dừng ngay, không charge, không tạo invoice. Nếu subscription vẫn active tại thời điểm khóa, job tiếp tục charge theo giá đã khóa dù ngay sau đó khách có đổi gói, việc đổi gói chỉ ghi vào `pending_plan_id` để có hiệu lực từ chu kỳ kế tiếp.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as SUBSCRIPTION + INVOICE store
    participant Gateway as Payment Gateway

    Job->>DB: Đọc SUBSCRIPTION S lúc 00:00:05 (plan=Pro, status=active, version=7)
    Job->>DB: UPDATE RENEWAL_JOB_ITEM SET status=locked, subscription_version_at_start=7

    Note over Job,DB: Khách bấm hủy subscription lúc 00:00:06, UPDATE SUBSCRIPTION SET status=cancelled, version=8

    Job->>DB: Kiểm tra lại SUBSCRIPTION.version ngay trước khi charge (đọc version=8, khác version=7 đã khóa)

    alt Version đã đổi (subscription bị hủy giữa chừng)
        Job->>DB: UPDATE RENEWAL_JOB_ITEM SET status=skipped_cancelled
        Job->>Job: Dừng ngay, không gọi cổng thanh toán, không tạo invoice
    else Version vẫn khớp (subscription chưa bị đổi)
        Job->>Gateway: Gọi charge thẻ theo giá gói Pro đã khóa tại thời điểm job bắt đầu
        Gateway-->>Job: Charge thành công
        Job->>DB: Ghi INVOICE(plan=Pro, idempotency_key=S-2026-09, status=paid), UPDATE SUBSCRIPTION current_period tiếp theo, RENEWAL_JOB_ITEM status=success

        Note over Job,DB: Nếu ngay lúc này khách bấm đổi gói Pro sang Basic, giao dịch charge đang gọi cổng thanh toán vẫn hoàn tất theo Pro, đổi gói chỉ ghi vào pending_plan_id=Basic có hiệu lực từ chu kỳ kế tiếp, không hủy giữa chừng request đang chờ phản hồi cổng thanh toán
    end
```
