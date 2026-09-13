# Enhance ERD — Optimistic lock theo nhóm trường, pessimistic lock cho chuyển giao, audit log chi tiết

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (1 `version` chung cho toàn bộ record), thay đổi và bổ sung:

- Tách `CUSTOMER_PROFILE` thành 3 nhóm trường độc lập, mỗi nhóm có `version` riêng: `CUSTOMER_GENERAL_INFO` (thông tin chung, vd phone), `CUSTOMER_SALE_INFO` (deal_stage), `CUSTOMER_CSKH_INFO` (contact_notes) — đáp ứng yêu cầu 1 (loại bỏ xung đột giả) và là nền tảng cho yêu cầu 2 (vẫn phát hiện đúng xung đột thật khi cùng sửa 1 trường trong cùng 1 nhóm).
- Entity mới `CUSTOMER_HANDOVER_LOCK` (locked_by, lock_token, expires_at) — pessimistic lock có timeout ngắn cho thao tác chuyển giao khách hàng, tách biệt hoàn toàn với cơ chế optimistic mặc định, đáp ứng yêu cầu 3.
- Entity mới `CUSTOMER_FIELD_CHANGE_LOG` (field_group, field_name, old_value, new_value) — ghi lịch sử chi tiết theo từng trường, phục vụ yêu cầu 4 (chỉ rõ đúng trường/nhóm trường bị người khác thay đổi) và yêu cầu 5 (audit đầy đủ cho xử lý khiếu nại).

```mermaid
erDiagram
    CUSTOMER ||--|| CUSTOMER_GENERAL_INFO : "có 1"
    CUSTOMER ||--|| CUSTOMER_SALE_INFO : "có 1"
    CUSTOMER ||--|| CUSTOMER_CSKH_INFO : "có 1"
    CUSTOMER ||--o{ CUSTOMER_HANDOVER_LOCK : "có lịch sử lock chuyển giao"
    CUSTOMER ||--o{ CUSTOMER_FIELD_CHANGE_LOG : "có lịch sử thay đổi từng trường"
    EMPLOYEE ||--o{ CUSTOMER_FIELD_CHANGE_LOG : "thực hiện thay đổi"

    CUSTOMER {
        string id PK
        string name
        string status
    }
    CUSTOMER_GENERAL_INFO {
        string customer_id PK,FK
        string phone
        string address
        int version "version riêng nhóm thông tin chung"
        string updated_by FK
        datetime updated_at
    }
    CUSTOMER_SALE_INFO {
        string customer_id PK,FK
        string deal_stage
        int version "version riêng nhóm sale"
        string updated_by FK
        datetime updated_at
    }
    CUSTOMER_CSKH_INFO {
        string customer_id PK,FK
        string contact_notes
        int version "version riêng nhóm CSKH"
        string updated_by FK
        datetime updated_at
    }
    CUSTOMER_HANDOVER_LOCK {
        string id PK
        string customer_id FK
        string locked_by FK
        string lock_token
        datetime acquired_at
        datetime expires_at "TTL ngắn, vd vài giây"
        string status "active|released|expired"
    }
    CUSTOMER_FIELD_CHANGE_LOG {
        string id PK
        string customer_id FK
        string field_group "general|sale|cskh"
        string field_name
        string old_value
        string new_value
        string changed_by FK
        datetime changed_at
    }
    EMPLOYEE {
        string id PK
        string name
        string role "sale|cskh"
    }
```
