# Sequence Diagram — Enhance: Search Product

Đây là **enhance**, flow "tìm kiếm sản phẩm" đã tồn tại ở base ([3-base-search-product-sqd.md](3-base-search-product-sqd.md)) nay thay đổi cốt lõi: chỉ 1 query duy nhất trên unified index thay vì query song song nhiều index riêng, và điểm khớp được chuẩn hóa theo đặc thù từng category trước khi so sánh, đảm bảo xếp hạng công bằng xuyên category.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Marketplace App
    participant Search as Search Service
    participant Index as Unified Product Index

    Buyer->>WebApp: Tìm "quà tặng sinh nhật"
    WebApp->>Search: Search request (keyword)

    Search->>Index: Query 1 lần duy nhất trên unified index, xuyên mọi category
    Index-->>Search: Kết quả kèm điểm khớp thô theo từng category

    Search->>Search: Chuẩn hóa điểm khớp theo đặc thù từng category (category_normalized_score) trước khi so sánh
    Search->>Search: Xếp hạng công bằng dựa trên điểm đã chuẩn hóa, không so trực tiếp điểm thô

    Search-->>WebApp: Kết quả xuyên category đã xếp hạng công bằng
    WebApp-->>Buyer: Hiển thị cả điện tử, thời trang, đồ chơi... xen kẽ hợp lý
```
