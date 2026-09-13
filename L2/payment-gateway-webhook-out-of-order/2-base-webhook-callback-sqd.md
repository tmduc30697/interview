# Base sequence — Xử lý webhook đơn giản (không idempotent, không xét thứ tự thật)

Đây là **base**, flow "Nhận webhook từ cổng thanh toán" ở trạng thái hiện tại — app cập nhật thẳng `ORDER.status`/`PAYMENT_TRANSACTION.status` theo nội dung webhook vừa nhận, không kiểm tra đã xử lý `transaction_id` này chưa, không xét thứ tự thật của sự kiện phía cổng thanh toán mà chỉ dựa vào thứ tự app nhận được. Flow này liên quan mật thiết tới enhance vì toàn bộ 2 yêu cầu đầu (idempotent theo transaction_id, xác định đúng thứ tự thật) đều nhằm sửa lỗ hổng ở đây.

```mermaid
sequenceDiagram
    participant Gateway as Cổng thanh toán
    participant App as Order Service
    participant DB as ORDER + PAYMENT_TRANSACTION store

    Gateway->>App: Webhook payment_success (transaction_id=TX-1)
    App->>DB: UPDATE PAYMENT_TRANSACTION SET status=success, ORDER SET status=paid
    App-->>Gateway: 200 OK

    Note over Gateway,App: Cổng thanh toán không nhận được response kịp thời, coi như thất bại và retry

    Gateway->>App: Webhook payment_success (transaction_id=TX-1) - lần retry 1
    App->>DB: UPDATE PAYMENT_TRANSACTION SET status=success, ORDER SET status=paid (chạy lại toàn bộ side-effect, vd cộng điểm thưởng, gửi email xác nhận lần 2)
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-1) - lần retry 2
    App->>DB: UPDATE lại lần 3, side-effect chạy lại lần 3
    App-->>Gateway: 200 OK

    Note over Gateway,App: Sau đó, webhook payment_refunded (xảy ra thật lúc T2) đến trước, rồi webhook payment_success cũ hơn (xảy ra thật lúc T1 < T2) đến sau do độ trễ mạng khác biệt

    Gateway->>App: Webhook payment_refunded (transaction_id=TX-2)
    App->>DB: UPDATE ORDER SET status=refunded
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-2, xảy ra thật trước refunded nhưng đến sau)
    App->>DB: UPDATE ORDER SET status=paid (ghi đè theo thời điểm nhận, không theo thời điểm xảy ra thật)
    App-->>Gateway: 200 OK

    Note over DB: Đơn hàng kết thúc ở trạng thái "paid" dù thực tế đã được hoàn tiền — sai vì dùng thời điểm nhận webhook thay vì thứ tự thật từ cổng thanh toán
```
