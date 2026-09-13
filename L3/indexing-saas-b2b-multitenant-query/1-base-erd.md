# ERD — Base (trước khi chuẩn hoá index đa tenant)

Đây là **base**: mô hình dữ liệu suy luận cho hệ thống SaaS B2B multi-tenant dùng shared schema *trước khi* áp các quy tắc tối ưu index của đề bài. Đề bài giả định đã tồn tại `TENANT`, `TENANT_USER` và một bảng dữ liệu nghiệp vụ dùng chung `RECORD` (đại diện cho loại bảng "shared table" bất kỳ — invoice, lead, ticket... — có cột `tenant_id`, quy mô lệch lớn giữa các tenant) — nếu không có các entity này thì việc "tối ưu index cho truy vấn đa tenant" sẽ không có nghĩa. ERD còn đưa `TABLE_INDEX` vào như metadata catalog (kiểu `pg_indexes`) để thể hiện đúng vấn đề gốc: ở base, thứ tự cột trong composite index KHÔNG được chuẩn hoá nhất quán — cột `leads_with_tenant_id` có giá trị hỗn hợp tuỳ theo index được tạo bởi tính năng nào, dẫn tới rủi ro full table scan nêu trong đề bài.

```mermaid
erDiagram
    TENANT ||--o{ TENANT_USER : employs
    TENANT ||--o{ RECORD : owns
    TENANT_USER ||--o{ RECORD : "creates/updates"
    RECORD ||--o{ TABLE_INDEX : "has index defined on (catalog-level, not per-row)"

    TENANT {
        string tenant_id PK
        string company_name
        string plan_tier
        datetime created_at
    }

    TENANT_USER {
        string user_id PK
        string tenant_id FK
        string email
        string role
        datetime created_at
    }

    RECORD {
        string record_id PK
        string tenant_id FK
        string owner_user_id FK
        string title
        string status
        decimal amount
        datetime created_at
        datetime updated_at
    }

    TABLE_INDEX {
        string index_id PK
        string table_name
        string index_name
        string column_order
        string index_type
        boolean leads_with_tenant_id
    }
```
