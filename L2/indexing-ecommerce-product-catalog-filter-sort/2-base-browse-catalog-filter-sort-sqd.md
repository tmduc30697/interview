# Base sequence - Browse catalog với filter/sort cơ bản

Đây là trạng thái **base**. Flow này mô tả cách user browse catalog, kết hợp filter (category, brand, price) và sort (giá tăng dần) cùng phân trang OFFSET, khi DB chưa có composite index hay keyset pagination. Flow này được chọn vì nó chính là hiện trạng "chưa tối ưu" mà đề bài yêu cầu cải thiện: DB phải scan/filesort và OFFSET quét bỏ dòng ở trang sâu.

```mermaid
sequenceDiagram
    actor User
    participant WebApp as Storefront Web/App
    participant CatalogAPI as Catalog Service
    participant DB as Product DB (Postgres)

    User->>WebApp: Chọn category + brand + khoảng giá, sort theo giá tăng dần, sang trang 5
    WebApp->>CatalogAPI: GET /products?category=..&brand=..&price_min=..&price_max=..&sort=price_asc&page=5
    CatalogAPI->>DB: SELECT * FROM products WHERE category_id=? AND brand_id=? AND price BETWEEN ? AND ? ORDER BY price ASC OFFSET 100 LIMIT 20
    Note over DB: Chỉ có index đơn cột trên category_id, không có composite index cho tổ hợp category+brand+price
    DB->>DB: Lọc category_id bằng index, sau đó lọc brand_id và price bằng scan tuần tự trên tập kết quả
    DB->>DB: Sắp xếp theo price bằng filesort vì không có index nào hỗ trợ ORDER BY này
    DB->>DB: OFFSET 100, quét và bỏ qua 100 dòng đầu trước khi lấy 20 dòng tiếp theo
    DB-->>CatalogAPI: 20 rows
    CatalogAPI-->>WebApp: Danh sách sản phẩm trang 5
    WebApp-->>User: Hiển thị kết quả
```

Điểm nghẽn ở đây (thiếu composite index cho filter/sort phổ biến, filesort, OFFSET quét bỏ dòng ở trang sâu) chính là 3 vấn đề đề bài nêu ra cần giải quyết bằng index và keyset pagination.
