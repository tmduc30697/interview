# ERD — Enhance (sau khi hợp nhất index đa category)

Đây là **enhance**: thay vì mỗi category 1 index riêng (base), nay chỉ còn 1 `PRODUCT_INDEX` thống nhất với schema thuộc tính động qua `PRODUCT_FACET_VALUE` (mô hình EAV), mỗi category định nghĩa facet của mình qua `CATEGORY_FACET_SCHEMA` mà không cần đổi cấu trúc index. Thêm `SEARCH_BEHAVIOR_LOG` phục vụ suy luận ngữ cảnh category cho autocomplete từ khóa mơ hồ.

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : classifies
    CATEGORY ||--o{ CATEGORY_FACET_SCHEMA : defines
    PRODUCT ||--o| PRODUCT_INDEX_DOCUMENT : "indexed into single unified index"
    PRODUCT_INDEX_DOCUMENT ||--o{ PRODUCT_FACET_VALUE : "has dynamic facet values"
    CATEGORY_FACET_SCHEMA ||--o{ PRODUCT_FACET_VALUE : "constrains"
    USER ||--o{ SEARCH_BEHAVIOR_LOG : generates

    CATEGORY {
        string category_id PK
        string name
    }

    PRODUCT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        string attributes_json
    }

    CATEGORY_FACET_SCHEMA {
        string schema_id PK
        string category_id FK
        string facet_key
        string facet_type
    }

    PRODUCT_INDEX_DOCUMENT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        decimal category_normalized_score
    }

    PRODUCT_FACET_VALUE {
        string facet_value_id PK
        string product_id FK
        string facet_key
        string facet_value
    }

    USER {
        string user_id PK
    }

    SEARCH_BEHAVIOR_LOG {
        string log_id PK
        string user_id FK
        string query_term
        string selected_category_id
        datetime searched_at
    }
```
