# Enhance sequence — Xem chi tiết đơn hàng đa seller (chỉ thấy phần của mình)

Đây là **enhance**, cùng flow "Xem chi tiết đơn hàng" đã có ở base nhưng nay thay đổi theo yêu cầu 2 của đề bài: mỗi seller tham gia một đơn hàng chỉ xem được `ORDER_ITEM` (số lượng, giá, vận chuyển) thuộc về sản phẩm của chính mình, không thấy toàn bộ đơn hàng gộp như base.

```mermaid
sequenceDiagram
    actor SellerA as Seller A
    participant App as Order Detail Service
    participant DB as ORDER / ORDER_ITEM store

    SellerA->>App: Xem chi tiết đơn hàng O (có sản phẩm của Seller A và Seller B)
    App->>DB: SELECT * FROM order_item WHERE order_id = O AND seller_id = A
    DB-->>App: Chỉ ORDER_ITEM thuộc Seller A trong đơn O
    App-->>SellerA: Hiển thị đúng phần của Seller A, kèm mã đơn hàng chung để đối chiếu với khách mua nhưng không có chi tiết phần của Seller B
    Note over App,DB: Nếu đơn hàng có nhiều seller, mỗi seller chỉ nhận view riêng của phần mình, không có API nào trả gộp toàn bộ item cho một seller thường
```
