# Enhance sequence - Browse catalog với filter/sort được index hỗ trợ

Đây là trạng thái **enhance** của flow browse-catalog-filter-sort. So với base, flow này thay đổi ở chỗ: Catalog Service giờ biết phân biệt tổ hợp sort nào được composite index hỗ trợ trực tiếp (price, nằm cuối `idx_cat_brand_price`) và tổ hợp nào không (bán chạy, rating) để chọn đường đi phù hợp — dùng partial index nếu có filter còn hàng, hoặc chấp nhận filesort chậm hơn cho tổ hợp hiếm, đúng chiến lược "vài composite index phổ biến nhất + fallback cho tổ hợp hiếm" mà đề bài yêu cầu.

```mermaid
sequenceDiagram
    actor User
    participant WebApp as Storefront Web/App
    participant CatalogAPI as Catalog Service
    participant DB as Product DB (Postgres)

    User->>WebApp: Chọn category + brand + khoảng giá, sort theo giá tăng dần
    WebApp->>CatalogAPI: GET /products?category=..&brand=..&price_min=..&price_max=..&sort=price_asc
    CatalogAPI->>CatalogAPI: Kiểm tra sort=price nằm trong idx_cat_brand_price, đường nhanh
    CatalogAPI->>DB: SELECT id,name,stock_qty FROM products WHERE category_id=? AND brand_id=? AND price BETWEEN ? AND ? ORDER BY price ASC LIMIT 20
    Note over DB: Index scan trực tiếp trên idx_cat_brand_price, không cần filesort
    Note over DB: Covering index chứa sẵn id/name/stock_qty, không cần đọc thêm heap page
    DB-->>CatalogAPI: 20 rows
    CatalogAPI-->>WebApp: Danh sách sản phẩm

    User->>WebApp: Đổi sort sang "bán chạy nhất"
    WebApp->>CatalogAPI: GET /products?category=..&sort=best_seller&in_stock=true
    CatalogAPI->>CatalogAPI: sort=best_seller không nằm trong idx_cat_brand_price, kiểm tra fallback
    alt Có filter còn hàng (in_stock=true)
        CatalogAPI->>DB: SELECT ... FROM products WHERE category_id=? AND stock_qty > 0 ORDER BY sold_count DESC LIMIT 20
        Note over DB: Dùng idx_cat_sold_partial, nhanh vì tổ hợp category+in-stock+bestseller đủ phổ biến để có index
        DB-->>CatalogAPI: 20 rows, độ trễ thấp
    else Không filter còn hàng, tổ hợp hiếm
        CatalogAPI->>DB: SELECT ... FROM products WHERE category_id=? ORDER BY sold_count DESC LIMIT 20
        Note over DB: Không có index phù hợp, filesort toàn bộ tập category, chấp nhận chậm hơn
        DB-->>CatalogAPI: 20 rows, độ trễ cao hơn
    end
    CatalogAPI-->>WebApp: Danh sách sản phẩm theo bán chạy
```
