# ERD — Enhance (sau khi có bảng lịch sử trạng thái)

Đây là **enhance**: ERD base cộng với `SHIPMENT_STATUS_EVENT` lưu đầy đủ mọi mốc trạng thái, khóa idempotent theo mã sự kiện từ đối tác, và cơ chế khóa/checkpoint cho backfill. So với base, cột `status` trên `SHIPMENT` không đổi ý nghĩa (vẫn là trạng thái mới nhất) nhưng nay được suy ra từ `SHIPMENT_STATUS_EVENT` có `event_timestamp` lớn nhất, thay vì chỉ đơn giản là giá trị ghi đè cuối cùng theo thứ tự xử lý.

```mermaid
erDiagram
    CARRIER_PARTNER ||--o{ SHIPMENT : handles
    SHIPMENT ||--o{ SHIPMENT_STATUS_EVENT : "has full history"
    SHIPMENT ||--o| BACKFILL_LOCK : "may be locked during backfill"

    CARRIER_PARTNER {
        string partner_id PK
        string name
        string integration_type
    }

    SHIPMENT {
        string shipment_id PK
        string partner_id FK
        string tracking_code
        string status
        datetime status_updated_at
    }

    SHIPMENT_STATUS_EVENT {
        string event_id PK
        string shipment_id FK
        string external_event_id
        string status
        string source
        datetime event_timestamp
        datetime received_at
    }

    BACKFILL_LOCK {
        string lock_id PK
        string shipment_id FK
        datetime acquired_at
        datetime released_at
    }
```
