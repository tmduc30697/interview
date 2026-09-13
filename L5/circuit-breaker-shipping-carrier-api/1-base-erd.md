# Base ERD — Hệ thống tạo vận đơn qua API carrier trước khi có circuit breaker

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có circuit breaker riêng biệt theo carrier. Đề bài giả định hệ thống logistics đã gọi API của nhiều đối tác vận chuyển để tạo vận đơn và tra cứu trạng thái, nên base cần đủ: danh sách carrier, đơn hàng, vận đơn, và log các lần gọi API (không phân loại lỗi, không breaker, không fallback, không đo lường riêng theo carrier — những thứ đó là phần enhance).

```mermaid
erDiagram
    ORDER ||--|| SHIPMENT : "generates"
    CARRIER ||--o{ SHIPMENT : "fulfills"
    SHIPMENT ||--o{ SHIPMENT_CALL_LOG : "logs calls"

    ORDER {
        string id PK
        string status "pending | shipped | failed"
    }
    CARRIER {
        string id PK
        string name
        string region
        decimal base_cost
    }
    SHIPMENT {
        string id PK
        string order_id FK
        string carrier_id FK
        string tracking_code
        string status "pending | created | failed"
    }
    SHIPMENT_CALL_LOG {
        string id PK
        string shipment_id FK
        string call_type "create | track"
        string status "success | failed | timeout"
        datetime called_at
    }
```
