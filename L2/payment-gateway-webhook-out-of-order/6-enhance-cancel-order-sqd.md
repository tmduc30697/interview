# Enhance sequence — Hủy đơn: webhook success là nguồn sự thật, chuyển sang hoàn tiền

Đây là **enhance**, flow "Khách bấm Hủy đơn" sau khi có quy tắc phân định rõ ai thắng khi đụng webhook thanh toán. So với base (last-write-wins), flow này thay đổi ở chỗ: trước khi hủy, app kiểm tra `PAYMENT_TRANSACTION.status` — nếu webhook success đã áp dụng (tiền đã thực sự bị trừ ở cổng thanh toán, nguồn sự thật), đơn hàng **không được hủy trực tiếp** mà chuyển sang `refund_pending` và khởi tạo luồng hoàn tiền, thay vì đảo trạng thái đơn giản theo request đến sau.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Order Service
    participant DB as ORDER + PAYMENT_TRANSACTION store
    participant Gateway as Cổng thanh toán

    par Khách bấm hủy đơn
        Customer->>App: Hủy đơn hàng #123
        App->>DB: Đọc PAYMENT_TRANSACTION.status hiện tại của đơn #123
    and Gần như đồng thời, webhook payment_success được áp dụng (theo flow ở file 5-enhance-webhook-callback-sqd.md)
        Gateway->>App: Webhook payment_success (order #123)
        App->>DB: UPDATE PAYMENT_TRANSACTION status=success, ORDER status=paid
    end

    alt PAYMENT_TRANSACTION.status vẫn là pending tại thời điểm đọc (chưa có webhook success nào áp dụng)
        App->>DB: UPDATE ORDER SET status=cancelled
        App-->>Customer: "Đơn hàng đã hủy"
    else PAYMENT_TRANSACTION.status đã là success (tiền đã bị trừ thật ở cổng thanh toán)
        App->>DB: UPDATE ORDER SET status=refund_pending
        App->>Gateway: Khởi tạo yêu cầu hoàn tiền
        App-->>Customer: "Đơn đã thanh toán, đang xử lý hoàn tiền thay vì hủy trực tiếp"
    end

    Note over App,DB: Webhook success luôn là nguồn sự thật — không đảo trạng thái đơn hàng chỉ vì request hủy ghi xuống sau
```
