# Base sequence — Xem chi tiết đơn hàng đa seller (lộ toàn bộ đơn hàng)

Đây là **base**, flow "Xem chi tiết đơn hàng" ở trạng thái hiện tại — khi một đơn hàng có sản phẩm từ nhiều seller, seller nào mở đơn hàng cũng thấy toàn bộ chi tiết gộp, kể cả phần của seller khác. Flow này liên quan mật thiết tới enhance vì yêu cầu 2 của đề bài chính là sửa đúng lỗ hổng này.

```mermaid
sequenceDiagram
    actor SellerA as Seller A
    participant App as Order Detail Service
    participant DB as ORDER / ORDER_ITEM store

    SellerA->>App: Xem chi tiết đơn hàng O (có sản phẩm của Seller A và Seller B)
    App->>DB: SELECT * FROM order_item WHERE order_id = O
    DB-->>App: Toàn bộ ORDER_ITEM của đơn O, gồm cả của Seller B
    App-->>SellerA: Hiển thị toàn bộ đơn hàng, gồm số lượng/giá/vận chuyển của cả Seller B
    Note over App,DB: Seller A nhìn thấy giá bán, số lượng, thông tin vận chuyển của sản phẩm thuộc Seller B — lộ dữ liệu kinh doanh của đối thủ
```
