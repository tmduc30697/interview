# Sequence Diagram — Enhance: Search Product

Đây là **enhance**, flow "tìm kiếm sản phẩm" đã tồn tại ở base ([2-base-search-product-sqd.md](2-base-search-product-sqd.md)) nay thay đổi hoàn toàn: query chuyển sang chạy trên `PRODUCT_INDEX_DOCUMENT` thay vì DB chính, autocomplete trả kết quả dưới 100ms, ưu tiên sản phẩm còn hàng/bán tốt, và xử lý được lỗi chính tả/đồng nghĩa/dấu tiếng Việt.

```mermaid
sequenceDiagram
    actor Customer
    participant WebApp as Storefront
    participant Search as Search Service
    participant Index as Product Search Index

    Customer->>WebApp: Gõ "tivi" (không dấu, có thể là "tv" hoặc "ti vi")
    WebApp->>Search: Autocomplete request (keyword, partial)

    Search->>Search: Chuẩn hóa từ khóa, tra bảng đồng nghĩa (tivi = tv = ti vi)
    Search->>Index: Query trên name_normalized + name_synonyms, filter stock_status
    Index-->>Search: Danh sách khớp, kèm popularity_score

    Search->>Search: Xếp hạng ưu tiên sản phẩm còn hàng và bán tốt lên trước
    Search-->>WebApp: Kết quả autocomplete (dưới 100ms)
    WebApp-->>Customer: Hiển thị gợi ý

    Note over Search,Index: Toàn bộ truy vấn chạy trên index, không đụng tới database chính, nên nhanh và ổn định dù có hàng triệu sản phẩm
```
