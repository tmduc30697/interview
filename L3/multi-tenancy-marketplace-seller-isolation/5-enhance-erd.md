# Enhance ERD — sau khi siết cách ly seller ở tầng dữ liệu

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 3 nhóm thay đổi/entity mới, ứng trực tiếp với yêu cầu 3, 4, 5 trong đề bài (yêu cầu 1 và 2 là thay đổi ở tầng enforcement/query logic trên đúng các entity đã có sẵn — `ORDER_ITEM.seller_id` vẫn là chỗ dựa để lọc, không cần entity mới):

- `SELLER` (sửa) — mở rộng `status` thêm `suspended | offboarded` và `deactivated_at`, phục vụ yêu cầu 4.
- `INDUSTRY_BENCHMARK_REPORT` (sửa) — thêm `seller_count_in_sample`, `min_sample_size`, `suppressed` để chặn suy luận ngược khi cỡ mẫu quá nhỏ (yêu cầu 3), đồng thời vẫn tính đúng khi có seller đã offboard (yêu cầu 4).
- `SELLER_OFFBOARDING_EVENT` (mới) — ghi nhận lý do và thời điểm seller rời sàn, phục vụ thu hồi quyền truy cập trong khi giữ lại dữ liệu lịch sử cho hỗ trợ khách hàng/pháp lý.
- `SUPPORT_ORDER_LOOKUP_LOG` (mới) — log mọi lần nhân viên chăm sóc khách hàng tra cứu một đơn hàng cụ thể, phục vụ yêu cầu 5.

```mermaid
erDiagram
    SELLER ||--o{ PRODUCT : lists
    BUYER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "ordered as"
    SELLER ||--o{ ORDER_ITEM : sells
    INDUSTRY_BENCHMARK_REPORT ||--o{ ORDER_ITEM : "computed from (cùng category, có ngưỡng mẫu tối thiểu)"
    SELLER ||--o{ SELLER_OFFBOARDING_EVENT : "may have"
    ORDER ||--o{ SUPPORT_ORDER_LOOKUP_LOG : "looked up via"

    SELLER {
        string id PK
        string name
        string status "active | suspended | offboarded"
        datetime joined_at
        datetime deactivated_at "nullable"
    }
    BUYER {
        string id PK
        string name
        string email
    }
    PRODUCT {
        string id PK
        string seller_id FK
        string category
        string title
        decimal price
        int stock
    }
    ORDER {
        string id PK
        string buyer_id FK
        datetime created_at
        string status
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string product_id FK
        string seller_id FK "chỗ dựa để enforce ở tầng data layer"
        int quantity
        decimal unit_price
        string shipping_status
    }
    INDUSTRY_BENCHMARK_REPORT {
        string id PK
        string category
        decimal avg_revenue
        int seller_count_in_sample
        int min_sample_size "vd 5"
        boolean suppressed "true nếu seller_count_in_sample < min_sample_size"
        datetime computed_at
    }
    SELLER_OFFBOARDING_EVENT {
        string id PK
        string seller_id FK
        string reason "policy_violation | voluntary"
        datetime offboarded_at
        string data_retention_note
    }
    SUPPORT_ORDER_LOOKUP_LOG {
        string id PK
        string support_staff_id
        string order_id FK
        string reason
        datetime looked_up_at
    }
```
