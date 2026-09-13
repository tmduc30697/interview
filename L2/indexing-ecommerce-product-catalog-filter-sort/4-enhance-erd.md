# Enhance ERD - Catalog sau khi áp chiến lược index

Đây là trạng thái **enhance**, tức nguyên văn đề bài đã áp dụng lên base. Vẫn giữ 3 entity Category/Brand/Product, nhưng bảng `products` giờ được gắn một tập composite/partial/covering index được chọn lọc dựa trên tần suất filter/sort thực tế (không tạo tràn lan). Thêm 2 entity mới để phục vụ các fallback mà đề bài yêu cầu: `PRODUCT_SEARCH_DOC` (search index riêng cho free-text tên sản phẩm, khi B-tree không đáp ứng tốt) và `FILTER_USAGE_STAT` (dữ liệu phân tích tần suất tổ hợp filter, làm căn cứ chọn composite index nào nên tạo, tổ hợp nào chấp nhận chậm hơn).

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : "classifies"
    BRAND ||--o{ PRODUCT : "manufactures"
    PRODUCT ||--o| PRODUCT_SEARCH_DOC : "synced to"
    CATEGORY ||--o{ FILTER_USAGE_STAT : "tracked by"

    CATEGORY {
        bigint id PK
        string name
        bigint parent_id FK "self-reference, nullable"
    }

    BRAND {
        bigint id PK
        string name
    }

    PRODUCT {
        bigint id PK
        string name
        bigint category_id FK "trong idx_cat_brand_price, idx_cat_sold_partial, idx_cat_created_at"
        bigint brand_id FK "trong idx_cat_brand_price"
        numeric price "cột cuối idx_cat_brand_price, covering thêm id/name/stock_qty"
        numeric rating "không có index riêng, sort theo rating chấp nhận filesort hoặc fallback search engine"
        int stock_qty "predicate của partial index idx_cat_sold_partial (WHERE stock_qty > 0)"
        int sold_count "cột trong idx_cat_sold_partial, ORDER BY sold_count DESC"
        timestamp created_at "cột trong idx_cat_created_at (category_id, created_at DESC), phục vụ sort mới nhất và keyset"
        timestamp updated_at
    }

    PRODUCT_SEARCH_DOC {
        bigint product_id PK "cùng giá trị id với PRODUCT, đồng bộ near real-time hoặc batch"
        string name_tokens "inverted/full-text index, Elasticsearch-like"
        bigint category_id
        numeric price
        int stock_qty
        timestamp synced_at
    }

    FILTER_USAGE_STAT {
        bigint id PK
        string filter_combo_key "vd category+brand+price, category+sold_count"
        bigint category_id FK
        bigint hit_count
        timestamp window_start
    }
```

So với base: thêm 3 composite/partial index có mục tiêu (`idx_cat_brand_price`, `idx_cat_sold_partial`, `idx_cat_created_at`) thay vì index tràn lan cho mọi tổ hợp, thêm `PRODUCT_SEARCH_DOC` để tách free-text search khỏi B-tree, và thêm `FILTER_USAGE_STAT` làm căn cứ dữ liệu cho quyết định "index nào phổ biến, tổ hợp nào hiếm nên fallback".
