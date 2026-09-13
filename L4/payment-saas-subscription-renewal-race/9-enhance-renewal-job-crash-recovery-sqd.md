# Enhance sequence — Job crash giữa chừng, restart không charge lần 2

Đây là **enhance**, flow mới phát sinh từ enhance — xử lý đúng yêu cầu 5 của đề bài: job crash ngay sau khi đã charge thành công ở cổng thanh toán nhưng chưa kịp ghi nhận kết quả vào `INVOICE`, lần chạy lại cho cùng subscription trong cùng chu kỳ phải kiểm tra idempotency trước khi gọi cổng thanh toán lại, tránh charge lần 2.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job (lần chạy 1)
    participant Gateway as Payment Gateway
    participant DB as INVOICE store
    participant JobRetry as Renewal Billing Job (lần chạy lại)

    Job->>DB: Kiểm tra INVOICE với idempotency_key=S-2026-09, chưa tồn tại
    Job->>Gateway: Gọi charge thẻ subscription S theo giá gói Pro
    Gateway-->>Job: Charge thành công (nhưng job crash ngay sau khi nhận response, trước khi ghi INVOICE)

    Note over Job,DB: Job process chết đột ngột, chưa kịp UPDATE INVOICE/SUBSCRIPTION

    Note over JobRetry: Job được lên lịch chạy lại (do batch chưa hoàn tất hoặc bị retry tự động)

    JobRetry->>DB: Kiểm tra INVOICE với idempotency_key=S-2026-09 trước khi gọi cổng thanh toán

    alt idempotency_key đã tồn tại với status=paid
        DB-->>JobRetry: Đã có INVOICE(status=paid) từ lần chạy trước
        JobRetry->>JobRetry: Không gọi lại cổng thanh toán, chỉ đảm bảo SUBSCRIPTION.current_period được cập nhật đúng nếu lần trước cũng chưa kịp ghi
    else idempotency_key chưa tồn tại hoặc status=failed
        JobRetry->>Gateway: Query trạng thái giao dịch trước đó theo idempotency_key (nếu gateway hỗ trợ), hoặc thử charge với cùng idempotency_key để gateway tự dedupe
        Gateway-->>JobRetry: Trả về kết quả thật (đã charge thành công trước đó)
        JobRetry->>DB: Ghi INVOICE(idempotency_key=S-2026-09, status=paid) đúng 1 lần
    end

    Note over JobRetry,DB: Nhờ kiểm tra idempotency_key trước khi charge, khách không bị charge 2 lần dù job restart giữa chừng sau crash
```
