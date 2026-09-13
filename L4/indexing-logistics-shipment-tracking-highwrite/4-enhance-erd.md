# Enhance ERD - Tối ưu index cho vận đơn tần suất ghi cao

Đây là **enhance**: cùng 2 bảng `shipments` và `shipment_status_history` như base, nhưng bổ sung: partial index cho status "đang xử lý" (thay vì index toàn bộ trạng thái), composite index (shipment_id, changed_at) cho lịch sử, index riêng phục vụ truy vấn cảnh báo SLA, chiến lược tracking_code thân thiện với index khi insert, và tách partition nóng/lạnh cho lịch sử để archive dữ liệu cũ mà không ảnh hưởng index đang phục vụ dữ liệu gần đây.

```mermaid
erDiagram
    SHIPMENT {
        bigint id PK
        string tracking_code UK "unique index idx_shipment_tracking_code, sinh dang K-sortable theo time_bucket cong random suffix de giam phan manh khi insert"
        string status "partial index idx_shipment_active_status (status) WHERE status khac delivered va cancelled, phuc vu dashboard loc don dang xu ly"
        string origin
        string destination
        timestamp estimated_delivery "partial index idx_sla_alert (estimated_delivery) WHERE status khac delivered, phuc vu job canh bao SLA dinh ky"
        timestamp created_at
        timestamp updated_at
    }
    SHIPMENT_STATUS_HISTORY {
        bigint id PK
        bigint shipment_id FK "composite index idx_history_shipment_changed (shipment_id, changed_at)"
        string status
        string station_code
        timestamp changed_at "partition theo khoang thoi gian, giu partition gan day nho gon va nong"
        string note
    }
    SHIPMENT_STATUS_HISTORY_ARCHIVE {
        bigint id PK
        bigint shipment_id "cung mang shipment_id nhu partition nong, giu index (shipment_id, changed_at) rieng tren du lieu cu"
        string status
        string station_code
        timestamp changed_at
        string note
    }
    SHIPMENT ||--o{ SHIPMENT_STATUS_HISTORY : "lich su gan day, partition nong con index nho"
    SHIPMENT ||--o{ SHIPMENT_STATUS_HISTORY_ARCHIVE : "lich su cu da archive, partition lanh tach rieng"
```
