# Enhance - Flow trạm cập nhật trạng thái vận đơn

Đây là **enhance** của flow **update-shipment-status**. So với base, flow này thay đổi ở 3 điểm: (1) UPDATE status giờ tương tác với partial index idx_shipment_active_status - entry chỉ tồn tại khi status còn "đang xử lý" và tự động bị xoá khỏi index khi chuyển sang delivered, (2) INSERT lịch sử ghi vào composite index (shipment_id, changed_at) thay vì bảng không có index chuyên biệt, (3) vì shipment_id là cột dẫn đầu của composite index, các trạm khác nhau ghi vào các vị trí khác nhau trong B-tree, giảm tranh chấp trang nóng so với base.

```mermaid
sequenceDiagram
    participant StationA as Trạm A
    participant StationB as Trạm B
    participant API as Shipment Service
    participant DB as PostgreSQL

    par Trạm A cập nhật vận đơn X sang out_for_delivery
        StationA->>API: PATCH /shipments/{trackingCodeX}/status (out_for_delivery)
        API->>DB: UPDATE shipments SET status='out_for_delivery' WHERE tracking_code = X
        Note over DB: status cũ và mới đều thoả điều kiện WHERE của idx_shipment_active_status, chỉ cập nhật giá trị trong index, không thêm/xoá entry
        API->>DB: INSERT INTO shipment_status_history (shipment_id=X, status, station_code=A, changed_at=now())
        Note over DB: ghi vào composite index idx_history_shipment_changed (shipment_id, changed_at), vị trí ghi phụ thuộc shipment_id nên phân tán trong B-tree
    and Trạm B cập nhật vận đơn Y sang delivered
        StationB->>API: PATCH /shipments/{trackingCodeY}/status (delivered)
        API->>DB: UPDATE shipments SET status='delivered' WHERE tracking_code = Y
        Note over DB: status mới delivered không còn thoả điều kiện partial index, entry của Y bị xoá khỏi idx_shipment_active_status, index tự động nhỏ lại theo thời gian
        API->>DB: INSERT INTO shipment_status_history (shipment_id=Y, status, station_code=B, changed_at=now())
    end
    DB-->>API: OK cho cả 2 giao dịch
    API-->>StationA: 200 OK
    API-->>StationB: 200 OK
    Note over DB: shipment_id ngẫu nhiên giữa các trạm giúp ghi đồng thời rải đều trên nhiều trang của composite index, thay vì dồn vào một trang cuối như khi chỉ index theo changed_at một mình
```
