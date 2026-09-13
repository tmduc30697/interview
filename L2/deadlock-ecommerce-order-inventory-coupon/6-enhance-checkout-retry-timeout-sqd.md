# Enhance sequence — Retry với backoff và timeout tối đa cho checkout mùa sale

Đây là **enhance**, flow mới mô tả tình huống mùa sale: nhiều đơn hàng chứa sản phẩm trùng lặp một phần (đơn A có X, Y, Z; đơn B có Z, X) chạy trong vài giây cao điểm, vẫn có thể phát sinh deadlock dù đã chuẩn hóa thứ tự lock (do độ trễ, lock chờ chồng chéo nhiều đơn). Đáp ứng yêu cầu 3 của đề bài: retry tự động tối đa 3 lần với backoff, và giới hạn timeout 2 giây cho toàn bộ transaction checkout.

```mermaid
sequenceDiagram
    actor Cust as Khách hàng (checkout giờ cao điểm)
    participant App as Checkout Service
    participant DB as Database
    participant Log as DEADLOCK_LOG

    Cust->>App: Yêu cầu checkout
    App->>App: Bắt đầu đếm timeout_ms=2000 cho toàn bộ transaction
    App->>DB: BEGIN Order (retry_attempt=1), lock order->inventory asc->coupon
    DB-->>App: Lỗi deadlock (do nhiều order khác cùng chạm sản phẩm Z, X)
    App->>Log: Ghi DEADLOCK_LOG (order_id, blocking_order_id, locked_resource, retry_attempt=1)

    App->>App: Kiểm tra thời gian đã trôi qua < timeout_ms, backoff ngắn rồi retry
    App->>DB: BEGIN Order (retry_attempt=2)
    DB-->>App: Lỗi deadlock lần nữa
    App->>Log: Ghi DEADLOCK_LOG (retry_attempt=2)

    App->>App: Kiểm tra thời gian đã trôi qua vẫn < timeout_ms, backoff rồi retry
    App->>DB: BEGIN Order (retry_attempt=3)
    DB-->>App: Transaction thành công, COMMIT
    App-->>Cust: Checkout thành công

    Note over App,DB: Nếu tổng thời gian vượt quá timeout_ms=2000 trước khi hết 3 lần retry, App chủ động hủy transaction, trả lỗi "thử lại sau" thay vì để đơn hàng treo kéo dài, tránh ảnh hưởng các đơn khác trong hàng đợi retry
```
