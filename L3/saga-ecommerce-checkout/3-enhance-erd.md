# ERD — Enhance (sau khi có saga orchestration)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — định nghĩa từng bước saga và compensate tương ứng, persist saga state để resume khi crash, retry/escalate khi compensate thất bại, và event log để replay/audit. So với base, `ORDER` không đổi cấu trúc nhưng nay trạng thái của nó được dẫn dắt bởi `SAGA_INSTANCE` thay vì được set trực tiếp từ Order Service.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : contains
    ORDER_ITEM }o--|| INVENTORY_ITEM : reserves
    ORDER ||--o| PAYMENT_CHARGE : "paid via"
    ORDER ||--o| SHIPMENT : "shipped via"

    ORDER ||--|| SAGA_INSTANCE : "orchestrated by"
    SAGA_INSTANCE ||--o{ SAGA_STEP : consists_of
    SAGA_STEP ||--o{ SAGA_EVENT : logs
    SAGA_STEP |o--o| COMPENSATION_ATTEMPT : "may trigger"

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

    SAGA_INSTANCE {
        string saga_id PK
        string order_id FK
        string current_step
        string status
        datetime started_at
        datetime updated_at
    }

    SAGA_STEP {
        string step_id PK
        string saga_id FK
        string name
        string status
        int sequence_order
        datetime executed_at
    }

    COMPENSATION_ATTEMPT {
        string attempt_id PK
        string step_id FK
        int attempt_number
        string status
        datetime attempted_at
        datetime next_retry_at
    }

    SAGA_EVENT {
        string event_id PK
        string step_id FK
        string event_type
        string payload
        datetime created_at
    }
```
