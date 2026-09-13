# Base - Flow trạm cập nhật trạng thái vận đơn

Đây là **base**, sequence diagram cho flow **update-shipment-status**: một trạm trung chuyển ghi nhận thay đổi trạng thái cho vận đơn. Flow này được chọn vì đây chính là nguồn ghi tần suất cao mà đề bài tập trung tối ưu index xung quanh nó (partial index status, composite index history, tránh hot page khi nhiều trạm ghi đồng thời).

```mermaid
sequenceDiagram
    participant StationA as Trạm A
    participant StationB as Trạm B
    participant API as Shipment Service
    participant DB as PostgreSQL

    par Trạm A cập nhật vận đơn X
        StationA->>API: PATCH /shipments/{trackingCodeX}/status (out_for_delivery)
        API->>DB: UPDATE shipments SET status, updated_at WHERE tracking_code = X
        Note over DB: WHERE dùng unique index tracking_code để seek, không có index riêng theo status
        API->>DB: INSERT INTO shipment_status_history (shipment_id=X, status, station_code=A, changed_at)
        Note over DB: bảng history chỉ có PK tự tăng, insert rẻ nhưng chưa tối ưu cho truy vấn timeline sau này
    and Trạm B cập nhật vận đơn Y cùng lúc
        StationB->>API: PATCH /shipments/{trackingCodeY}/status (delivered)
        API->>DB: UPDATE shipments SET status, updated_at WHERE tracking_code = Y
        API->>DB: INSERT INTO shipment_status_history (shipment_id=Y, status, station_code=B, changed_at)
    end
    DB-->>API: OK cho cả 2 giao dịch
    API-->>StationA: 200 OK
    API-->>StationB: 200 OK
    Note over DB: chưa có cấu trúc index nào được thiết kế riêng để phân tán ghi đồng thời từ nhiều trạm, tiềm ẩn rủi ro tranh chấp trang khi tải tăng
```
