# ERD — Base (trước khi hợp nhất index đa category)

Đây là **base**: mô hình dữ liệu suy luận cho marketplace *trước khi* hợp nhất facet đa category. Đề bài chỉ ra điểm cần tránh là "phải tạo index riêng biệt cho từng category gây khó khăn khi tìm xuyên category" — suy ra base chính là trạng thái đó: mỗi category có 1 index riêng với schema thuộc tính cố định cho category đó, khiến tìm xuyên category khó khăn. Đây là tiền đề để yêu cầu "index thống nhất, schema thuộc tính động" có nghĩa.

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : classifies
    CATEGORY ||--|| CATEGORY_INDEX : "has its own separate index"
    PRODUCT ||--o| CATEGORY_INDEX_DOCUMENT : "indexed only into its category's index"

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

    CATEGORY_INDEX {
        string index_id PK
        string category_id FK
        string fixed_attribute_schema
    }

    CATEGORY_INDEX_DOCUMENT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        string category_specific_attributes
    }
```
