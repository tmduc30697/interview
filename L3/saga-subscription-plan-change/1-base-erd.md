# ERD — Base (trước khi có saga đổi gói)

Đây là **base**: mô hình dữ liệu suy luận cho SaaS B2B *trước khi* có saga điều phối đổi gói. Đề bài giả định hệ thống đã có billing, entitlement, và notification như 3 service riêng biệt, mỗi khách hàng có subscription và entitlement tương ứng gói hiện tại — nếu không có sẵn các entity này thì "saga đảm bảo billing và entitlement luôn khớp nhau" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ đổi gói, không suy diễn thêm các module không liên quan (support, usage analytics...).

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : has
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--|| ENTITLEMENT : grants
    SUBSCRIPTION ||--o{ BILLING_INVOICE : generates

    CUSTOMER {
        string customer_id PK
        string company_name
        string email
    }

    PLAN {
        string plan_id PK
        string name
        decimal monthly_price
        string feature_limits
    }

    SUBSCRIPTION {
        string subscription_id PK
        string customer_id FK
        string plan_id FK
        string status
    }

    ENTITLEMENT {
        string entitlement_id PK
        string subscription_id FK
        string enabled_features
        int user_limit
        string status
    }

    BILLING_INVOICE {
        string invoice_id PK
        string subscription_id FK
        decimal amount
        string status
        datetime created_at
    }
```
