# ERD — Enhance (sau khi có tìm kiếm theo bản đồ)

Đây là **enhance**: `LISTING` được bổ sung tọa độ `lat`/`lng`, thêm `LISTING_GEO_INDEX_DOCUMENT` phục vụ query vùng bản đồ hiệu quả, `ADDRESS_INDEX_ENTRY` phục vụ autocomplete địa chỉ chuẩn hóa, và `VIEWPORT_QUERY_CACHE` để trả kết quả nhanh khi zoom/pan liên tục. So với base, tìm kiếm không còn dựa trên `address_text` khớp chuỗi mà dựa trên tọa độ và index địa lý chuyên dụng.

```mermaid
erDiagram
    AGENT ||--o{ LISTING : posts
    LISTING ||--o| LISTING_GEO_INDEX_DOCUMENT : "indexed with coordinates"
    ADDRESS_INDEX_ENTRY ||--o{ LISTING : "resolves to"

    AGENT {
        string agent_id PK
        string full_name
        string phone
    }

    LISTING {
        string listing_id PK
        string agent_id FK
        string address_text
        decimal lat
        decimal lng
        decimal price
        string listing_type
        string status
        datetime created_at
    }

    LISTING_GEO_INDEX_DOCUMENT {
        string listing_id PK
        decimal lat
        decimal lng
        string geohash
        decimal price
        string status
        datetime indexed_at
    }

    ADDRESS_INDEX_ENTRY {
        string address_entry_id PK
        string normalized_address
        string alias_text
        decimal lat
        decimal lng
    }

    VIEWPORT_QUERY_CACHE {
        string cache_key PK
        string bounding_box
        string cached_result_ids
        datetime expires_at
    }
```
