# Base sequence — Hủy đơn hàng (last-write-wins, không xét nguồn sự thật)

Đây là **base**, flow "Khách bấm Hủy đơn" ở trạng thái hiện tại — app cập nhật thẳng `ORDER.status = cancelled` theo request của khách, không kiểm tra webhook thanh toán có đang/đã xử lý song song hay không. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 3 của đề bài quy định rõ ai thắng khi request hủy đơn chạm webhook success gần như đồng thời — mà base hiện chưa có quy tắc nào cả, ai ghi sau thắng.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Order Service
    participant DB as ORDER store
    participant Gateway as Cổng thanh toán

    par Khách bấm hủy đơn
        Customer->>App: Hủy đơn hàng #123
        App->>DB: UPDATE ORDER SET status=cancelled
    and Gần như đồng thời, cổng thanh toán báo đã trừ tiền thành công
        Gateway->>App: Webhook payment_success (order #123)
        App->>DB: UPDATE ORDER SET status=paid
    end

    Note over DB: Ai ghi sau thắng — nếu update "cancelled" chạy sau, đơn hàng hiển thị đã hủy dù tiền thực tế đã bị trừ ở cổng thanh toán, khách mất tiền mà tưởng đơn đã hủy
```
