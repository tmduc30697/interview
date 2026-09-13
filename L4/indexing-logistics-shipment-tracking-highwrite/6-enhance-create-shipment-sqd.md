# Enhance - Flow tạo vận đơn mới với tracking_code thân thiện index

Đây là **enhance**, sequence diagram cho flow mới **create-shipment**: tạo vận đơn và sinh mã tracking_code. Base đã có unique index trên tracking_code nhưng không mô tả cách sinh mã; flow này được thêm để làm rõ chiến lược sinh mã mới - vừa khó đoán vừa hạn chế phân mảnh index khi insert liên tục với tần suất cao, đúng yêu cầu đầu tiên của đề bài.

```mermaid
sequenceDiagram
    participant Client as Merchant/App tạo đơn
    participant API as Shipment Service
    participant DB as PostgreSQL

    Client->>API: POST /shipments (origin, destination, estimated_delivery)
    API->>API: Sinh tracking_code = time_bucket hiện tại (vd theo giờ) + chuỗi random 6 ký tự
    Note over API: time_bucket làm mã tăng dần thô theo thời gian nhưng vẫn đủ random trong từng bucket để khó đoán, khác với UUIDv4 hoàn toàn ngẫu nhiên
    API->>DB: INSERT INTO shipments (tracking_code, status='created', origin, destination, estimated_delivery)
    Note over DB: giá trị tracking_code mới rơi vào vùng gần cuối B-tree của unique index trong một khoảng thời gian ngắn, giảm page-split/phân mảnh so với insert hoàn toàn ngẫu nhiên trên toàn bộ không gian khoá
    Note over DB: trong từng time_bucket mã vẫn đủ random để khách hàng không đoán được mã liền kề của vận đơn khác
    DB-->>API: shipment_id, tracking_code
    API-->>Client: 201 Created (tracking_code)
```
