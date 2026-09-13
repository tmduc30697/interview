# Base ERD — Cache flash sale trước khi tối ưu cho tần suất thay đổi cực cao

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có các cơ chế đặc thù cho flash sale. Đề bài nói flash sale đã có lịch giảm giá theo giờ và giá/tồn kho hiển thị dựa vào cache — nên base cần đủ: sản phẩm (kèm lịch sale), 1 cache entry chung cho giá+tồn kho với TTL cố định, và đơn hàng. Chưa có entity nào phục vụ invalidation chủ động theo đơn hàng/bypass cache khi sắp hết hàng/pre-warm theo lịch/double-check giá — những thứ đó là phần enhance.

```mermaid
erDiagram
    PRODUCT ||--o| PRODUCT_CACHE_ENTRY : "cached as"
    PRODUCT ||--o{ ORDER : "ordered via"

    PRODUCT {
        string id PK
        string name
        decimal price
        decimal sale_price
        int stock
        datetime sale_start_at
        datetime sale_end_at
    }
    PRODUCT_CACHE_ENTRY {
        string cache_key PK "product_id"
        string value "price hiệu lực + stock"
        datetime cached_at
        int ttl_seconds "vd 30-60"
    }
    ORDER {
        string id PK
        string product_id FK
        int quantity
        decimal charged_price
        datetime created_at
    }
```
