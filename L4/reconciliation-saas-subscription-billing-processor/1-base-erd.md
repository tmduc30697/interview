# ERD — Base (trước khi có đối soát billing với processor)

Đây là **base**: mô hình dữ liệu suy luận cho SaaS thuê bao *trước khi* có flow đối soát. Đề bài giả định hệ thống đã có billing nội bộ phát sinh invoice (kể cả invoice điều chỉnh proration khi đổi gói) và đã tích hợp payment processor để thu tiền — nếu không có sẵn `INVOICE` và `PROCESSOR_CHARGE` thì "đối soát billing nội bộ với số tiền processor báo cáo" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ billing + thu tiền, không suy diễn thêm các module không liên quan (support ticket, usage metering chi tiết...).

```mermaid
erDiagram
    CUSTOMER ||--o{ SUBSCRIPTION : has
    PLAN ||--o{ SUBSCRIPTION : "subscribed to"
    SUBSCRIPTION ||--o{ INVOICE : generates
    INVOICE }o--o| PROCESSOR_CHARGE : "collected via"

    CUSTOMER {
        string customer_id PK
        string company_name
        string email
    }

    PLAN {
        string plan_id PK
        string name
        decimal monthly_price
    }

    SUBSCRIPTION {
        string subscription_id PK
        string customer_id FK
        string plan_id FK
        string status
        date current_period_start
        date current_period_end
    }

    INVOICE {
        string invoice_id PK
        string subscription_id FK
        string type
        decimal amount
        string currency
        string status
        datetime created_at
    }

    PROCESSOR_CHARGE {
        string charge_id PK
        string invoice_id FK
        string processor_reference_id
        decimal amount_charged
        string status
        datetime charged_at
    }
```
