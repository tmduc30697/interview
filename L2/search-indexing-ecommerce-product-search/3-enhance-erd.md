# ERD — Enhance (sau khi có search index)

Đây là **enhance**: ERD base cộng với `PRODUCT_INDEX_DOCUMENT` (tài liệu index phục vụ autocomplete/tìm kiếm, có các trường chuẩn hóa cho lỗi chính tả/đồng nghĩa/dấu tiếng Việt), `SYNC_EVENT` ghi nhận thay đổi cần đồng bộ, `INDEX_ALIAS` để swap giữa 2 index song song khi reindex, và `RECONCILIATION_RUN` đối soát định kỳ. So với base, `PRODUCT` không đổi cấu trúc nhưng mọi thay đổi giờ phát sinh `SYNC_EVENT` để đẩy vào index gần thời gian thực thay vì chỉ tồn tại trong DB chính.

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : classifies
    PRODUCT ||--o{ SYNC_EVENT : "triggers on change"
    SYNC_EVENT ||--o| PRODUCT_INDEX_DOCUMENT : "updates"
    PRODUCT ||--o| PRODUCT_INDEX_DOCUMENT : "mirrored into"
    INDEX_ALIAS ||--o{ PRODUCT_INDEX_DOCUMENT : "points to active index"
    RECONCILIATION_RUN ||--o{ PRODUCT : "samples for comparison"

    PRODUCT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        int stock_quantity
        datetime updated_at
    }

    CATEGORY {
        string category_id PK
        string name
    }

    PRODUCT_INDEX_DOCUMENT {
        string product_id PK
        string name_normalized
        string name_synonyms
        decimal price
        string stock_status
        decimal popularity_score
        string category_id
        datetime indexed_at
    }

    SYNC_EVENT {
        string event_id PK
        string product_id FK
        string change_type
        string status
        datetime created_at
        datetime processed_at
    }

    INDEX_ALIAS {
        string alias_id PK
        string active_index_name
        string standby_index_name
        datetime switched_at
    }

    RECONCILIATION_RUN {
        string run_id PK
        date run_date
        int sample_size
        int mismatch_count
        string status
    }
```
