# Sequence Diagram — Base: Index Product

Đây là **base**, flow "lập chỉ mục sản phẩm" khi mỗi category có 1 index riêng biệt — tiền đề cho thấy hạn chế: sản phẩm chỉ được index vào đúng 1 index của category nó thuộc về, theo schema cố định của category đó.

```mermaid
sequenceDiagram
    actor Seller
    participant ProductSvc as Product Service
    participant DB as Product Database
    participant Indexer as Category Index Worker

    Seller->>ProductSvc: Đăng sản phẩm mới (category = electronics)
    ProductSvc->>DB: INSERT INTO products (category_id, attributes_json)
    DB-->>ProductSvc: product_id created
    ProductSvc->>Indexer: Sync sang index của category electronics

    Indexer->>Indexer: Áp fixed_attribute_schema của category electronics
    Indexer-->>ProductSvc: Indexed vào electronics_index

    Note over Indexer: Sản phẩm chỉ tồn tại trong index riêng của category electronics, không nằm trong index chung nào để tìm xuyên category
```
