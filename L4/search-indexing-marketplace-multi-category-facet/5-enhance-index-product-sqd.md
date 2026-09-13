# Sequence Diagram — Enhance: Index Product

Đây là **enhance**, flow "lập chỉ mục sản phẩm" đã tồn tại ở base ([2-base-index-product-sqd.md](2-base-index-product-sqd.md)) nay thay đổi cốt lõi: sản phẩm được index vào duy nhất 1 `PRODUCT_INDEX` thống nhất, thuộc tính riêng của category được lưu dưới dạng `PRODUCT_FACET_VALUE` động tra theo `CATEGORY_FACET_SCHEMA`, không cần index riêng biệt cho từng category.

```mermaid
sequenceDiagram
    actor Seller
    participant ProductSvc as Product Service
    participant DB as Product Database
    participant Indexer as Unified Index Worker
    participant SchemaStore as Category Facet Schema
    participant Index as Unified Product Index

    Seller->>ProductSvc: Đăng sản phẩm mới (category = electronics, attributes_json)
    ProductSvc->>DB: INSERT INTO products (category_id, attributes_json)
    DB-->>ProductSvc: product_id created
    ProductSvc->>Indexer: Sync sang unified index

    Indexer->>SchemaStore: Lấy facet schema của category electronics (RAM, màn hình...)
    SchemaStore-->>Indexer: Danh sách facet_key hợp lệ cho electronics

    Indexer->>Index: Ghi PRODUCT_INDEX_DOCUMENT (name, price, category_id)
    Indexer->>Index: Ghi PRODUCT_FACET_VALUE cho từng thuộc tính theo schema động
    Index-->>Indexer: Indexed

    Note over Index: Sản phẩm nằm trong cùng 1 index với mọi category khác, tìm xuyên category không cần gộp nhiều nguồn
```
