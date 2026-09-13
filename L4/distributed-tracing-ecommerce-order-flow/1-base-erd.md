# ERD - Base (trước khi có distributed tracing)

Đây là trạng thái **base**: sàn e-commerce đã có luồng đặt hàng đi qua 4 service tách biệt (`cart-service`, `inventory-service`, `payment-service`, `order-confirmation-service`), có các entity nghiệp vụ cốt lõi là `ORDER`, `INVENTORY_HOLD` (giữ tồn kho) và `PAYMENT_TRANSACTION`, nhưng mỗi service chỉ ghi log cục bộ, không có định danh chung nối các bước lại với nhau. Đây là tiền đề khiến enhance cần thêm tracing để chỉ ra chính xác bước nào đang là nút thắt.

```mermaid
erDiagram
    ORDER ||--o| INVENTORY_HOLD : "giữ tồn kho cho"
    ORDER ||--o| PAYMENT_TRANSACTION : "thanh toán qua"
    ORDER ||--o{ SERVICE_LOG : "phát sinh log rải rác"

    ORDER {
        string order_id PK
        string cart_id
        string status
        datetime created_at
    }
    INVENTORY_HOLD {
        string hold_id PK
        string order_id FK
        string sku
        int quantity
        string status
    }
    PAYMENT_TRANSACTION {
        string payment_id PK
        string order_id FK
        decimal amount
        string gateway_reference
        string status
    }
    SERVICE_LOG {
        string log_id PK
        string service_name
        string level
        string message
        datetime logged_at
    }
```
