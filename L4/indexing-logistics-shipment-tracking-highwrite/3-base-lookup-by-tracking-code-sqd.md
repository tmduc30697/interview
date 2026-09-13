# Base - Flow khách tra cứu theo mã vận đơn

Đây là **base**, sequence diagram cho flow **lookup-by-tracking-code**: khách hàng tra cứu vận đơn bằng mã tracking_code. Flow này được chọn vì nó là lý do unique index trên `tracking_code` tồn tại ngay từ base, và là flow phải luôn giữ nhanh dù bảng bị ghi liên tục - chính là mối lo mà đề bài nêu ở yêu cầu sinh mã tracking_code thân thiện với index.

```mermaid
sequenceDiagram
    participant Customer as Khách hàng
    participant API as Shipment Service
    participant DB as PostgreSQL

    Customer->>API: GET /shipments/{tracking_code}
    API->>DB: SELECT * FROM shipments WHERE tracking_code = ?
    Note over DB: dùng unique index idx_shipment_tracking_code, hiện tại còn nhỏ nên tra cứu nhanh
    DB-->>API: thông tin vận đơn
    API-->>Customer: 200 OK

    Note over DB: mã tracking_code hiện sinh ngẫu nhiên không tuần tự, mỗi insert vận đơn mới rơi vào vị trí bất kỳ trong B-tree, càng nhiều insert index càng dễ phân mảnh, về lâu dài có thể làm chậm dần chính flow tra cứu này
```
