# Enhance sequence - Tìm kiếm từ khóa kết hợp filter

Đây là trạng thái **enhance**, mô tả 1 flow hoàn toàn mới phát sinh từ đề bài: chưa tồn tại ở base vì base chưa có nhu cầu free-text search, chỉ filter theo cột có cấu trúc. Flow này được chọn vì đề bài yêu cầu xác định ranh giới rõ ràng: khi user gõ từ khóa tên sản phẩm kết hợp filter, B-tree index không đáp ứng tốt free-text nên phải tách sang search index riêng (Elasticsearch-like), rồi hydrate dữ liệu mới nhất (giá, tồn kho) từ DB chính.

```mermaid
sequenceDiagram
    actor User
    participant WebApp as Storefront Web/App
    participant CatalogAPI as Catalog Service
    participant SearchEngine as Search Index (Elasticsearch-like)
    participant DB as Product DB (Postgres)

    User->>WebApp: Gõ "áo thun nam" và chọn thêm lọc khoảng giá
    WebApp->>CatalogAPI: GET /search?q=áo thun nam&price_min=..&price_max=..
    CatalogAPI->>CatalogAPI: Phát hiện có free-text keyword, B-tree index trên products không phù hợp
    CatalogAPI->>SearchEngine: Query full-text trên name_tokens, kết hợp filter price/category
    Note over SearchEngine: PRODUCT_SEARCH_DOC được đồng bộ near real-time hoặc batch từ bảng products
    SearchEngine-->>CatalogAPI: Danh sách product_id khớp, đã rank theo độ liên quan
    CatalogAPI->>DB: SELECT id,name,price,stock_qty FROM products WHERE id IN (...)
    Note over DB: Hydrate lại giá/tồn kho mới nhất, vì search index có thể lệch so với dữ liệu real-time
    DB-->>CatalogAPI: Chi tiết sản phẩm hiện tại
    CatalogAPI-->>WebApp: Kết quả tìm kiếm kết hợp filter
    WebApp-->>User: Hiển thị kết quả
```

Ranh giới rút ra: mọi truy vấn chỉ filter/sort trên cột có cấu trúc vẫn đi qua composite/partial index trên `products` như các flow browse-catalog-filter-sort, còn ngay khi có free-text keyword thì route hẳn sang Search Index, không cố gắng LIKE '%...%' trên B-tree.
