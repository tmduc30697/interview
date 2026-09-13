# ERD — Base (trước khi có bảng lịch sử trạng thái)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng logistics *trước khi* tách lịch sử trạng thái. Đề bài giả định đã có `SHIPMENT` với 1 cột `status` đơn, được nhiều `CARRIER_PARTNER` (đối tác vận chuyển) cập nhật qua webhook hoặc polling — nếu không có sẵn cột `status` đơn thì yêu cầu "chuyển sang bảng shipment_status_history" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ theo dõi trạng thái vận đơn, không suy diễn thêm định giá cước/thanh toán không liên quan.

```mermaid
erDiagram
    CARRIER_PARTNER ||--o{ SHIPMENT : handles
    SHIPMENT ||--o{ STATUS_UPDATE_LOG : "raw webhook log (unstructured, not queried)"

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

    STATUS_UPDATE_LOG {
        string log_id PK
        string shipment_id FK
        string raw_payload
        datetime received_at
    }
```
