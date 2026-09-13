# Enhance sequence — Server down khi nhận webhook, retry của cổng + polling dự phòng

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — xử lý tình huống server nội bộ down đúng lúc cổng thanh toán gửi webhook, đảm bảo đơn hàng không bị "treo" vĩnh viễn ở trạng thái `processing` chỉ vì lỡ mất đúng 1 webhook. Dựa vào 2 lớp phòng vệ: cơ chế retry sẵn có của cổng thanh toán, và job polling chủ động của app cho các đơn hàng bị kẹt lâu.

```mermaid
sequenceDiagram
    participant Gateway as Cổng thanh toán
    participant App as Order Service
    participant DB as ORDER + PAYMENT_TRANSACTION store
    participant Poller as Status Polling Job (định kỳ)

    Gateway->>App: Webhook payment_success (order #456)
    Note over App: Server nội bộ đang down, request không đến được / timeout
    Gateway->>Gateway: Không nhận được 200 OK, lên lịch retry theo cơ chế riêng của cổng thanh toán

    loop Cổng thanh toán tự retry theo backoff riêng của họ
        Gateway->>App: Webhook payment_success (order #456) - retry
    end

    alt Server hồi phục trước khi cổng thanh toán retry lần kế tiếp
        Gateway->>App: Webhook payment_success (order #456) - retry thành công
        App->>DB: Xử lý idempotent như flow 5-enhance-webhook-callback-sqd.md, UPDATE status=success
        App-->>Gateway: 200 OK
    else Cổng thanh toán hết lượt retry hoặc độ trễ quá lâu trước khi server hồi phục
        Note over Poller,DB: Job polling định kỳ quét các ORDER đang ở status=processing quá 1 khoảng thời gian ngưỡng
        Poller->>DB: Tìm ORDER #456 đang processing quá lâu
        Poller->>Gateway: Chủ động gọi API truy vấn trạng thái giao dịch theo gateway_transaction_id
        Gateway-->>Poller: Trạng thái thật = success
        Poller->>DB: Áp dụng cập nhật tương tự webhook (idempotent theo gateway_transaction_id), UPDATE status=success
    end

    Note over Poller,DB: Nhờ polling dự phòng, đơn hàng không bao giờ treo vĩnh viễn chỉ vì lỡ đúng 1 webhook lúc server down
```
