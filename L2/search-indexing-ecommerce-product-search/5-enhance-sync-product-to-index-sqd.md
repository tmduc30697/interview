# Sequence Diagram — Enhance: Sync Product to Index

Đây là **enhance**, flow hoàn toàn mới: khi sản phẩm đổi giá hoặc hết hàng trong database chính, thay đổi phải được phản ánh vào search index trong vòng vài giây, tránh hiển thị giá cũ hoặc để khách tìm thấy sản phẩm đã hết hàng mà không có cảnh báo.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên vận hành
    participant Admin as Admin/Inventory Service
    participant DB as Product Database
    participant Queue as Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Product Search Index

    Staff->>Admin: Cập nhật giá hoặc đánh dấu hết hàng
    Admin->>DB: UPDATE products SET price/stock_quantity
    DB-->>Admin: Updated
    Admin->>Queue: Publish SYNC_EVENT (product_id, change_type)

    Queue->>Indexer: Deliver event
    Indexer->>DB: Đọc lại dữ liệu mới nhất của product
    DB-->>Indexer: name, price, stock_quantity mới
    Indexer->>Index: Update PRODUCT_INDEX_DOCUMENT (price, stock_status)
    Index-->>Indexer: Indexed
    Indexer->>Queue: Mark event processed

    Note over Queue,Indexer: Toàn bộ chuỗi hoàn tất trong vòng vài giây kể từ khi DB được cập nhật
```
