# Enhance - Flow khách xem timeline lịch sử vận đơn

Đây là **enhance**, sequence diagram cho flow mới **view-shipment-timeline**: khách hàng xem toàn bộ lịch sử thay đổi trạng thái theo đúng thứ tự thời gian. Flow này hoàn toàn mới so với base (base chưa có index hỗ trợ truy vấn timeline hiệu quả), được thêm để minh hoạ composite index (shipment_id, changed_at) và chiến lược partition nóng/lạnh giải quyết yêu cầu thứ ba của đề bài.

```mermaid
sequenceDiagram
    participant Customer as Khách hàng
    participant API as Shipment Service
    participant DB as PostgreSQL partitioned history

    Customer->>API: GET /shipments/{tracking_code}/timeline
    API->>DB: SELECT id FROM shipments WHERE tracking_code = ?
    DB-->>API: shipment_id
    API->>DB: SELECT status, station_code, changed_at FROM shipment_status_history WHERE shipment_id = ? ORDER BY changed_at
    Note over DB: bảng shipment_status_history được partition theo changed_at (vd theo tháng), planner chỉ quét các partition liên quan thay vì toàn bảng
    Note over DB: composite index idx_history_shipment_changed (shipment_id, changed_at) trên từng partition trả đúng thứ tự timeline mà không cần sort riêng
    DB-->>API: danh sách các dòng trạng thái theo thứ tự thời gian
    API-->>Customer: 200 OK (timeline)

    Note over DB: partition cũ định kỳ được chuyển sang SHIPMENT_STATUS_HISTORY_ARCHIVE, giữ partition nóng phục vụ dữ liệu gần đây luôn nhỏ gọn và index không bị phình theo thời gian
```
