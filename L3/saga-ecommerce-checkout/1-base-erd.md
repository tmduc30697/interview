# ERD — Base (trước khi có saga orchestration)

Đây là **base**: mô hình dữ liệu suy luận cho checkout e-commerce *trước khi* có saga điều phối. Đề bài giả định hệ thống đã có `ORDER`, đã tích hợp Inventory/Payment/Shipping như 3 service riêng biệt — nếu không có sẵn các entity này thì "saga điều phối transaction xuyên 3 service" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ checkout, không suy diễn thêm các module không liên quan (khuyến mãi, đánh giá sản phẩm...).

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    ORDER_ITEM }o--|| INVENTORY_ITEM : reserves
    ORDER ||--o| PAYMENT_CHARGE : "paid via"
    ORDER ||--o| SHIPMENT : "shipped via"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        decimal total_amount
        string status
        datetime created_at
    }

    ORDER_ITEM {
        string order_item_id PK
        string order_id FK
        string sku
        int quantity
    }

    INVENTORY_ITEM {
        string sku PK
        int available_quantity
        int reserved_quantity
    }

    PAYMENT_CHARGE {
        string charge_id PK
        string order_id FK
        decimal amount
        string status
    }

    SHIPMENT {
        string shipment_id PK
        string order_id FK
        string status
        string tracking_code
    }
```
