# Enhance - Flow dashboard vận hành lọc vận đơn theo trạng thái

Đây là **enhance**, sequence diagram cho flow mới **filter-active-shipments-dashboard**: bộ phận vận hành lọc danh sách vận đơn đang xử lý. Flow này hoàn toàn mới so với base (base chưa có index nào phục vụ lọc theo status), được thêm để minh hoạ trực tiếp partial index idx_shipment_active_status giải quyết yêu cầu thứ hai của đề bài.

```mermaid
sequenceDiagram
    participant Operator as Nhân viên vận hành
    participant Dashboard as Ops Dashboard
    participant API as Shipment Service
    participant DB as PostgreSQL

    Operator->>Dashboard: Lọc danh sách vận đơn đang xử lý
    Dashboard->>API: GET /shipments?status=active
    API->>DB: SELECT * FROM shipments WHERE status NOT IN (delivered, cancelled) ORDER BY updated_at DESC
    Note over DB: query planner dùng partial index idx_shipment_active_status, index này chỉ chứa các dòng đang active nên kích thước nhỏ ổn định dù bảng shipments có hàng triệu đơn đã delivered từ lâu
    DB-->>API: danh sách vận đơn active
    API-->>Dashboard: 200 OK (list)
    Dashboard-->>Operator: hiển thị bảng lọc theo trạng thái
```
