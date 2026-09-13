# Base sequence — Checkout payment (gọi trực tiếp, retry mù quáng)

Đây là **base**, flow "Thanh toán lúc checkout" ở trạng thái hiện tại — gọi thẳng gateway, retry ngay lập tức không phân biệt loại lỗi, không kiểm tra idempotency. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng các lỗ hổng ở đây.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant Gateway as Payment Gateway (bên thứ ba)
    participant DB as PAYMENT_ATTEMPT store

    Customer->>App: Xác nhận thanh toán
    App->>Gateway: Gọi API charge
    Gateway-->>App: Timeout (không rõ đã charge hay chưa)
    App->>DB: Ghi PAYMENT_ATTEMPT(status=timeout)
    App->>Gateway: Retry ngay lập tức (không kiểm tra idempotency, không phân biệt lỗi có an toàn để retry hay không)
    Gateway-->>App: Charge thành công (lần 2)
    Note over Gateway,DB: Nếu lần gọi đầu thực ra đã charge thành công phía gateway trước khi timeout trả về, khách hàng đã bị charge 2 lần
    App->>DB: Ghi PAYMENT_ATTEMPT(status=success)
    App-->>Customer: "Thanh toán thành công"
    Note over App,Gateway: Khi gateway đang chậm/lỗi hàng loạt, mọi request checkout đều tự retry dồn dập, làm gateway càng quá tải hơn
```
