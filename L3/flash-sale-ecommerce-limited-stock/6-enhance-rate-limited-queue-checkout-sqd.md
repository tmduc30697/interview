# Sequence - Enhance: Hàng đợi/rate-limiter chặn trước khi chạm DB

Đây là flow **enhance hoàn toàn mới**, mô tả lớp hàng đợi/rate-limiter đặt trước transaction trừ tồn kho. Khi hàng chục nghìn request cùng đổ vào lúc 12:00:00, chỉ 1 số lượng giới hạn/giây được cho phép chạm vào DB, các request còn lại nhận ngay phản hồi "đang xử lý, vui lòng chờ" thay vì để tất cả cùng đấm vào DB gây timeout hàng loạt. Đáp ứng **yêu cầu 2** của đề bài.

```mermaid
sequenceDiagram
    participant Users as Hàng chục nghìn khách
    participant GW as API Gateway / Rate Limiter
    participant Queue as QUEUE_TICKET Store
    participant API as Checkout API
    participant DB as Database

    Users->>GW: Hàng chục nghìn request "Mua ngay" gần như cùng lúc
    GW->>Queue: Tạo QUEUE_TICKET(status=waiting) cho từng request
    GW-->>Users: Phản hồi ngay "Đang xử lý, vui lòng chờ" kèm ticket_id, không block request

    loop Rate limiter chỉ cho phép N request/giây chạm DB
        GW->>Queue: Lấy ticket có enqueued_at sớm nhất, status=waiting
        Queue-->>GW: ticket_id kế tiếp
        GW->>API: Cho phép ticket này chạy transaction trừ tồn kho
        API->>DB: UPDATE stock_qty - 1 WHERE stock_qty > 0
        DB-->>API: affected_rows
        API->>Queue: Cập nhật ticket status=done, kèm kết quả (mua được / hết hàng)
    end

    Users->>GW: Poll kết quả bằng ticket_id
    GW->>Queue: Tra trạng thái ticket
    Queue-->>GW: status=done, kết quả=mua được / hết hàng
    GW-->>Users: Trả kết quả cuối cùng
    Note over DB: DB chỉ nhận đúng N request/giây, không bị hàng chục nghìn kết nối đồng thời làm timeout hàng loạt
```
