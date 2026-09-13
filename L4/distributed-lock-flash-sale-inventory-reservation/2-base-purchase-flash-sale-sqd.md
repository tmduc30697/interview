# Base sequence — Đọc-trừ tồn kho trực tiếp, oversell khi mua đồng thời

Đây là **base**, flow mua hàng hiện tại: server đọc `stock_quantity`, kiểm tra còn hàng rồi trừ đi trong 1 transaction DB thông thường, không có khóa nào bảo vệ khoảng giữa đọc và ghi. Đây chính là kịch bản oversell nêu trong đề bài — hàng nghìn request mua hàng gửi tới cùng lúc khi mở bán flash sale, nhiều request cùng đọc thấy còn hàng rồi cùng trừ.

```mermaid
sequenceDiagram
    actor ReqA as Request A
    actor ReqB as Request B
    participant App as Order Service
    participant DB as Database

    Note over App,DB: Sản phẩm chỉ còn 1 đơn vị tồn kho

    ReqA->>App: Mua sản phẩm SKU-123, số lượng 1
    ReqB->>App: Mua sản phẩm SKU-123, số lượng 1 (gần như cùng lúc)

    App->>DB: (A) SELECT stock_quantity FROM PRODUCT WHERE sku=SKU-123
    DB-->>App: (A) stock_quantity = 1

    App->>DB: (B) SELECT stock_quantity FROM PRODUCT WHERE sku=SKU-123
    DB-->>App: (B) stock_quantity = 1

    Note over App: Cả 2 request đều đọc thấy còn hàng, chưa có khóa chặn giữa đọc và ghi

    App->>DB: (A) UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    App->>DB: (A) INSERT ORDER (product_sku=SKU-123, quantity=1, status=confirmed)

    App->>DB: (B) UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    App->>DB: (B) INSERT ORDER (product_sku=SKU-123, quantity=1, status=confirmed)

    DB-->>App: Cả 2 UPDATE/INSERT đều thành công
    App-->>ReqA: Mua thành công
    App-->>ReqB: Mua thành công

    Note over DB: stock_quantity bị trừ về -1, 2 đơn hàng được tạo cho 1 sản phẩm duy nhất, oversell
```
