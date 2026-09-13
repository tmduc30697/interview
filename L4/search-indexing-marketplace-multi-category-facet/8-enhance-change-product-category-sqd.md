# Sequence Diagram — Enhance: Change Product Category

Đây là **enhance**, flow hoàn toàn mới: khi người bán đổi category của sản phẩm sau khi đã đăng, index phải cập nhật đồng thời danh mục lẫn bộ facet thuộc tính tương ứng, đảm bảo sản phẩm không còn xuất hiện ở facet của category cũ.

```mermaid
sequenceDiagram
    actor Seller
    participant ProductSvc as Product Service
    participant DB as Product Database
    participant SchemaStore as Category Facet Schema
    participant Indexer as Unified Index Worker
    participant Index as Unified Product Index

    Seller->>ProductSvc: Đổi category sản phẩm (electronics -> furniture)
    ProductSvc->>DB: UPDATE products SET category_id = furniture
    DB-->>ProductSvc: Updated
    ProductSvc->>Indexer: Sync thay đổi category

    Indexer->>Index: Xóa toàn bộ PRODUCT_FACET_VALUE cũ theo schema electronics
    Index-->>Indexer: Facet cũ đã gỡ, sản phẩm không còn khớp bộ lọc electronics

    Indexer->>SchemaStore: Lấy facet schema mới của category furniture
    SchemaStore-->>Indexer: Danh sách facet_key hợp lệ cho furniture (chất liệu, kích thước...)

    Indexer->>Index: Cập nhật category_id trên PRODUCT_INDEX_DOCUMENT, ghi PRODUCT_FACET_VALUE mới theo schema furniture
    Index-->>Indexer: Indexed

    Note over Index: Việc xóa facet cũ và ghi facet mới xảy ra trong cùng 1 lần sync, không để sản phẩm lơ lửng ở cả 2 category cùng lúc
```
