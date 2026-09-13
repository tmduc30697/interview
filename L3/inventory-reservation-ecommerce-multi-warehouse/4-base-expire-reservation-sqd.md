# Sequence - Base - Flow "expire-reservation"

Đây là **base**: cron job giải phóng reservation hết hạn bằng cách blind-decrement cột `reserved`, dựa trên `reserved_at` của `CartItem`, không kiểm tra gì thêm. Flow này được chọn vì nó là tiền đề cho yêu cầu 3 - khi job này chạy gần như đồng thời với request thanh toán, không có cơ chế nào đảm bảo thứ tự đúng, có thể vừa charge tiền vừa đã giải phóng tồn kho cho người khác.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)
    participant ExpireJob as Cron Expire Job

    Note over Inventory: CartItem.reserved_at cách đây 15 phút, reserved = 1

    par Request thanh toán tới gần như đồng thời
        Customer->>App: xác nhận thanh toán
        App->>Inventory: trừ tồn kho thật, available -= 1
        App-->>Customer: thanh toán thành công, charge tiền
    and Cron job chạy đúng lúc đó
        ExpireJob-->>ExpireJob: quét CartItem có reserved_at quá 15 phút
        ExpireJob->>Inventory: reserved -= 1 (giải phóng, không kiểm tra thanh toán có đang chạy không)
    end

    Note over Inventory: Không rõ thứ tự nào xảy ra trước,\ncó thể vừa charge tiền khách vừa giải phóng tồn kho cho người khác giữ tiếp
```
