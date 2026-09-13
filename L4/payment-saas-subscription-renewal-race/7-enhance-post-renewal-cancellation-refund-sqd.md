# Enhance sequence — Hủy ngay sau khi charge thành công, áp chính sách hoàn tiền

Đây là **enhance**, flow mới phát sinh từ enhance — khách bấm hủy subscription 5 giây sau khi job billing đã charge thành công chu kỳ mới. Xử lý đúng yêu cầu 3 của đề bài: áp chính sách hoàn tiền theo tỷ lệ (hoặc không hoàn) cho phần thời gian còn lại của chu kỳ, và đảm bảo trạng thái subscription cuối cùng nhất quán, không mâu thuẫn giữa "đã charge chu kỳ mới" và "đã hủy ngay lập tức".

```mermaid
sequenceDiagram
    participant Job as Renewal Billing Job
    participant DB as SUBSCRIPTION + INVOICE store
    actor Customer
    participant App as Subscription Service
    participant Refund as REFUND store

    Job->>DB: Charge thành công chu kỳ mới, ghi INVOICE(status=paid), UPDATE SUBSCRIPTION current_period_start=hôm nay, current_period_end=+30 ngày

    Note over Customer,App: 5 giây sau

    Customer->>App: Bấm hủy subscription
    App->>DB: Đọc SUBSCRIPTION, thấy INVOICE mới nhất status=paid, billed_at cách đây 5 giây

    App->>App: Áp chính sách hoàn tiền theo tỷ lệ cho phần thời gian còn lại của chu kỳ vừa charge (gần như toàn bộ 30 ngày)
    App->>Refund: Ghi REFUND(invoice_id, refund_type=prorated, amount=tính theo số ngày còn lại)
    App->>DB: UPDATE SUBSCRIPTION SET status=cancelled, current_period_end=hôm nay (kết thúc chu kỳ ngay, không giữ hiệu lực tới hết 30 ngày)

    App-->>Customer: "Đã hủy, đã hoàn tiền theo tỷ lệ cho phần thời gian chưa dùng"

    Note over DB,Refund: Trạng thái cuối cùng nhất quán: subscription = cancelled, invoice chu kỳ mới vẫn giữ status=paid (đúng sự thật đã charge), phần chênh lệch được thể hiện qua REFUND thay vì sửa lại INVOICE
```
