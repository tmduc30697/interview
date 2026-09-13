# Base ERD - Quản lý vận đơn cơ bản

Đây là **base**: trạng thái dữ liệu trước khi áp đề bài tối ưu index. Hệ thống đã có bảng `shipments` với unique index trên `tracking_code` để tra cứu tức thời, và bảng `shipment_status_history` lưu lịch sử thay đổi trạng thái, nhưng **chưa có** partial index cho status, chưa có composite index (shipment_id, changed_at) cho lịch sử, và chưa có index riêng phục vụ truy vấn cảnh báo SLA - đây chính là những khoảng trống mà enhance sẽ lấp vào.

```mermaid
erDiagram
    SHIPMENT {
        bigint id PK
        string tracking_code UK "unique btree index idx_shipment_tracking_code, dung tra cuu khach hang"
        string status "khong co index rieng, filter theo status phai quet toan bang"
        string origin
        string destination
        timestamp estimated_delivery "khong co index, chua ho tro truy van SLA hieu qua"
        timestamp created_at
        timestamp updated_at
    }
    SHIPMENT_STATUS_HISTORY {
        bigint id PK
        bigint shipment_id FK "chi co FK, khong co index composite theo (shipment_id, changed_at)"
        string status
        string station_code "ma tram trung chuyen thuc hien cap nhat"
        timestamp changed_at
        string note
    }
    SHIPMENT ||--o{ SHIPMENT_STATUS_HISTORY : "co nhieu dong lich su trang thai"
```
