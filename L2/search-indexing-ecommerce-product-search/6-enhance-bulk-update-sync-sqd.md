# Sequence Diagram — Enhance: Bulk Update Sync

Đây là **enhance**, flow hoàn toàn mới: khi có cập nhật giá hàng loạt dịp sale lớn ảnh hưởng hàng trăm nghìn sản phẩm, pipeline đồng bộ index phải xử lý mà không làm chậm hoặc downtime cho tìm kiếm đang phục vụ người dùng thật.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên vận hành
    participant Admin as Admin/Inventory Service
    participant DB as Product Database
    participant Queue as Sync Event Queue
    participant Indexer as Index Sync Worker (bulk lane)
    participant Index as Product Search Index

    Staff->>Admin: Trigger bulk price update (sale lớn, hàng trăm nghìn sản phẩm)
    Admin->>DB: Batch UPDATE products SET price theo từng lô nhỏ
    Admin->>Queue: Publish SYNC_EVENT hàng loạt, gắn priority = bulk

    Queue->>Indexer: Route các event bulk vào lane riêng, tách khỏi lane sync thời gian thực
    loop for each batch of events
        Indexer->>Index: Bulk update PRODUCT_INDEX_DOCUMENT theo lô, throttled
        Index-->>Indexer: Batch indexed
    end

    Note over Queue,Indexer: Lane bulk chạy song song nhưng có giới hạn tốc độ riêng, không cạnh tranh tài nguyên với lane sync thời gian thực của các sản phẩm khác
    Indexer-->>Admin: Bulk sync completed, tìm kiếm vẫn phục vụ liên tục trong suốt quá trình
```
