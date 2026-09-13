# Base ERD — Cache trang sản phẩm chỉ dựa vào TTL cố định

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có invalidation theo event. Đề bài nói cache trang chi tiết sản phẩm (giá, tồn kho, mô tả) với TTL 1 giờ — nên base cần đủ: sản phẩm và 1 cache entry theo product_id với TTL cố định dài. Chưa có entity nào phục vụ invalidation theo event/khoá chống stampede/đo lường — những thứ đó là phần enhance.

```mermaid
erDiagram
    PRODUCT ||--o| PRODUCT_CACHE_ENTRY : "cached as"

    PRODUCT {
        string id PK
        string name
        decimal price
        int stock
        string description
        datetime updated_at
    }
    PRODUCT_CACHE_ENTRY {
        string cache_key PK "product_id"
        string value "price + stock + description"
        datetime cached_at
        int ttl_seconds "3600"
    }
```
