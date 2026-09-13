# Enhance ERD — sau khi tối ưu cache cho flash sale

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- Tách `PRICE_CACHE_ENTRY` và `INVENTORY_CACHE_ENTRY` riêng (thay vì gộp chung như base) — tồn kho cần TTL rất ngắn + invalidate chủ động, còn giá chủ yếu đổi theo lịch nên có thể pre-warm.
- `INVENTORY_INVALIDATION_EVENT` (mới) — phát sinh ngay khi có đơn hàng trừ kho.
- `PRODUCT` thêm `low_stock_threshold` — sản phẩm dưới ngưỡng này bỏ qua cache, đọc thẳng DB.
- `SALE_WARMUP_JOB` (mới) — pre-warm cache trước thời điểm đổi giá theo lịch vài giây, chống stampede.
- `CHECKOUT_PRICE_CHECK` (mới) — double-check giá ở backend trước khi charge.
- `DISPLAY_ACCURACY_LOG` (mới) — log các lần hiển thị sai giá/tồn kho để làm baseline đo lường.

```mermaid
erDiagram
    PRODUCT ||--o| PRICE_CACHE_ENTRY : "cached as"
    PRODUCT ||--o| INVENTORY_CACHE_ENTRY : "cached as"
    PRODUCT ||--o{ INVENTORY_INVALIDATION_EVENT : triggers
    PRODUCT ||--o{ SALE_WARMUP_JOB : "scheduled for"
    PRODUCT ||--o{ ORDER : "ordered via"
    PRODUCT ||--o{ DISPLAY_ACCURACY_LOG : "measured on"
    ORDER ||--o| CHECKOUT_PRICE_CHECK : "verified by"
    ORDER ||--o| INVENTORY_INVALIDATION_EVENT : "causes"

    PRODUCT {
        string id PK
        string name
        decimal price
        decimal sale_price
        int stock
        int low_stock_threshold
        datetime sale_start_at
        datetime sale_end_at
    }
    PRICE_CACHE_ENTRY {
        string cache_key PK "product_id"
        decimal price_value
        datetime cached_at
        int ttl_seconds
    }
    INVENTORY_CACHE_ENTRY {
        string cache_key PK "product_id"
        int stock_value
        datetime cached_at
        int ttl_seconds "rất ngắn, vài giây"
    }
    INVENTORY_INVALIDATION_EVENT {
        string id PK
        string product_id FK
        string order_id FK
        datetime triggered_at
    }
    SALE_WARMUP_JOB {
        string id PK
        string product_id FK
        datetime scheduled_change_at
        datetime warmup_started_at
        string status
    }
    ORDER {
        string id PK
        string product_id FK
        int quantity
        decimal displayed_price
        decimal charged_price
        datetime created_at
    }
    CHECKOUT_PRICE_CHECK {
        string id PK
        string order_id FK
        decimal displayed_price
        decimal authoritative_price
        boolean match
        datetime checked_at
    }
    DISPLAY_ACCURACY_LOG {
        string id PK
        string product_id FK
        string metric_type "price_mismatch | stock_oversell_risk"
        string displayed_value
        string actual_value
        datetime detected_at
    }
```
