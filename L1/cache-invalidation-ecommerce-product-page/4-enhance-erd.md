# Enhance ERD — sau khi có invalidation theo event + chống stampede

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `PRODUCT` thêm `version` — dùng để xây snapshot cache atomic (giá + tồn kho cùng 1 version), tránh trạng thái nửa cũ nửa mới.
- `PRODUCT_CACHE_ENTRY` thêm `source_version` và `ttl_seconds` rút ngắn (dự phòng) — kết hợp cả invalidate theo event lẫn TTL ngắn thay vì chỉ dựa TTL dài như base.
- `CACHE_INVALIDATION_EVENT` (mới) — phát sinh ngay khi giá/tồn kho đổi, invalidate trong vài giây.
- `SINGLEFLIGHT_LOCK` (mới) — chỉ 1 request được rebuild cache khi miss, chống cache stampede.
- `CACHE_METRIC` + `STALENESS_METRIC` (mới) — đo hit ratio theo loại trang và độ trễ invalidate thực tế.

```mermaid
erDiagram
    PRODUCT ||--o| PRODUCT_CACHE_ENTRY : "cached as"
    PRODUCT ||--o{ CACHE_INVALIDATION_EVENT : triggers
    PRODUCT ||--o{ STALENESS_METRIC : measures
    PRODUCT_CACHE_ENTRY ||--o| SINGLEFLIGHT_LOCK : "guarded by (khi rebuild)"

    PRODUCT {
        string id PK
        string name
        decimal price
        int stock
        string description
        int version
        datetime updated_at
    }
    PRODUCT_CACHE_ENTRY {
        string cache_key PK "product_id"
        string value "price + stock + description (snapshot atomic theo version)"
        int source_version
        datetime cached_at
        int ttl_seconds "30-60, dự phòng"
    }
    CACHE_INVALIDATION_EVENT {
        string id PK
        string product_id FK
        string changed_fields "price | stock | both"
        int source_version
        datetime triggered_at
    }
    SINGLEFLIGHT_LOCK {
        string id PK
        string cache_key
        string holder_request_id
        datetime acquired_at
        datetime expires_at
    }
    CACHE_METRIC {
        string id PK
        string page_type "product_detail"
        datetime window_start
        datetime window_end
        int hit_count
        int miss_count
    }
    STALENESS_METRIC {
        string id PK
        string product_id FK
        datetime changed_at
        datetime cache_consistent_at
        int staleness_ms
    }
```
