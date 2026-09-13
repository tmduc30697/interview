# ERD — Enhance (sau khi có tìm kiếm ngữ nghĩa)

Đây là **enhance**: thêm `ARTICLE_EMBEDDING` (vector ngữ nghĩa) và `ARTICLE_INDEX_DOCUMENT` (index tìm kiếm riêng, tách khỏi DB nguồn) để trả kết quả nhanh mà vẫn xếp hạng theo ngữ nghĩa, `ARTICLE_VERSION` cho các phiên bản theo gói/sản phẩm khác nhau, và `REINDEX_BATCH` cho sửa hàng loạt. So với base, tìm kiếm không còn query trực tiếp DB mà qua `ARTICLE_INDEX_DOCUMENT`, và trạng thái draft/removed được lọc cứng ngay ở tầng ghi index chứ không chỉ ở tầng query.

```mermaid
erDiagram
    AUTHOR ||--o{ ARTICLE : writes
    ARTICLE ||--o| ARTICLE_EMBEDDING : "has semantic vector"
    ARTICLE ||--o{ ARTICLE_VERSION : "has plan-specific versions"
    ARTICLE ||--o| ARTICLE_INDEX_DOCUMENT : "mirrored into (published only)"
    REINDEX_BATCH ||--o{ ARTICLE_INDEX_DOCUMENT : "updates atomically"

    AUTHOR {
        string author_id PK
        string full_name
    }

    ARTICLE {
        string article_id PK
        string author_id FK
        string title
        string content
        string status
        datetime updated_at
    }

    ARTICLE_EMBEDDING {
        string embedding_id PK
        string article_id FK
        string vector
        datetime generated_at
    }

    ARTICLE_VERSION {
        string version_id PK
        string article_id FK
        string product_plan
        string content_override
    }

    ARTICLE_INDEX_DOCUMENT {
        string article_id PK
        string title
        string content
        string vector
        string product_plan
        string status
        datetime indexed_at
    }

    REINDEX_BATCH {
        string batch_id PK
        string reason
        string status
        datetime started_at
        datetime completed_at
    }
```
