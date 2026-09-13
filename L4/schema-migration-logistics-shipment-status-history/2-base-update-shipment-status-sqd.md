# Sequence Diagram — Base: Update Shipment Status

Đây là **base**, flow "cập nhật trạng thái vận đơn" — tiền đề bắt buộc cho việc tách bảng lịch sử: chính flow này ghi đè trực tiếp lên cột `status` của `SHIPMENT` mỗi khi có cập nhật, không lưu vết các mốc trạng thái trước đó — đây chính là hạn chế mà enhance phải giải quyết.

```mermaid
sequenceDiagram
    participant Partner as Carrier Partner
    participant Webhook as Webhook Handler
    participant DB as Shipments Table

    Partner->>Webhook: Push status update (shipment_id, new_status, event_time)
    Webhook->>DB: UPDATE shipments SET status = new_status, status_updated_at = now()
    DB-->>Webhook: Updated
    Webhook-->>Partner: Ack received

    Note over DB: Trạng thái cũ bị ghi đè hoàn toàn, không còn lưu vết lịch sử các mốc trước đó
```
