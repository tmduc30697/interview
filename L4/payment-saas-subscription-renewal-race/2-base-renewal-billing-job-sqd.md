# Base sequence — Job renewal charge thẻ (không lock, không version check)

Đây là **base**, flow "Job billing tự động renewal" ở trạng thái hiện tại — job đọc gói hiện tại của subscription, gọi cổng thanh toán charge theo gói đó, rồi ghi invoice và cập nhật chu kỳ mới, không kiểm tra lại subscription có bị đổi/hủy giữa chừng hay không, không có idempotency key. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu 1, 2 và 5 của đề bài đều nhằm sửa đúng các lỗ hổng race và crash-recovery ở đây.

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as SUBSCRIPTION + INVOICE store
    participant Gateway as Payment Gateway

    Job->>DB: Đọc SUBSCRIPTION S (00:00:05, plan=Pro, status=active)
    Note over Job: Không lock, không ghi nhận version tại thời điểm đọc

    Job->>Gateway: Gọi charge thẻ theo giá gói Pro
    Gateway-->>Job: Charge thành công

    Job->>DB: Ghi INVOICE(plan=Pro, status=paid), UPDATE SUBSCRIPTION current_period tiếp theo

    Note over Job,DB: Nếu khách bấm hủy subscription lúc 00:00:06, ngay giữa lúc job đang gọi cổng thanh toán, job vẫn không biết và vẫn hoàn tất charge + tạo invoice cho gói đã bị hủy
```
