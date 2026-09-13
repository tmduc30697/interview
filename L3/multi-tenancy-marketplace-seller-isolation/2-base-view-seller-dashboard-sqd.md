# Base sequence — Xem dashboard người bán (lọc ở tầng UI, không phải tầng data)

Đây là **base**, flow "Xem dashboard người bán" ở trạng thái hiện tại — API nội bộ dùng chung cho nhiều mục đích của nền tảng trả về dữ liệu chưa được giới hạn chặt theo `seller_id`, việc chỉ hiển thị đúng dữ liệu của seller đang đăng nhập được xử lý ở tầng giao diện. Flow này liên quan mật thiết tới enhance vì yêu cầu 1 của đề bài chính là chuyển việc giới hạn này xuống tầng thấp nhất thay vì dựa vào UI.

```mermaid
sequenceDiagram
    actor Seller as Seller A
    participant App as Seller Dashboard App
    participant Agg as Internal Aggregate API (dùng chung nhiều mục đích)
    participant DB as ORDER_ITEM store

    Seller->>App: Xem đơn hàng/doanh thu/tồn kho của mình
    App->>Agg: Gọi API tổng hợp số liệu theo category/thời gian
    Agg->>DB: Query ORDER_ITEM trong khoảng thời gian, theo category (không lọc seller_id ở query)
    DB-->>Agg: Trả về toàn bộ ORDER_ITEM liên quan, gồm cả của seller khác
    Agg-->>App: Trả về tập dữ liệu chưa giới hạn theo seller
    App->>App: Lọc lại ở tầng giao diện, chỉ hiển thị hàng có seller_id = Seller A
    App-->>Seller: Hiển thị dashboard (đã lọc ở UI)
    Note over App,Agg: Nếu UI lọc sai hoặc client tự gọi thẳng Internal Aggregate API, dữ liệu seller khác bị lộ vì tầng dưới không hề enforce seller_id
```
