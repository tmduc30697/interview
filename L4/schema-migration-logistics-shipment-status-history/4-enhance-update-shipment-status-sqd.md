# Sequence Diagram — Enhance: Update Shipment Status

Đây là **enhance**, flow "cập nhật trạng thái vận đơn" đã tồn tại ở base ([2-base-update-shipment-status-sqd.md](2-base-update-shipment-status-sqd.md)) nay thay đổi lớn: mỗi cập nhật phải insert thêm 1 dòng vào `SHIPMENT_STATUS_EVENT` (dual-write cùng transaction với cột `status` cũ), cột `status` chỉ được ghi đè nếu `event_timestamp` mới lớn hơn timestamp hiện tại (không phải thời điểm hệ thống nhận được), và webhook trùng lặp phải idempotent theo `external_event_id`.

```mermaid
sequenceDiagram
    participant PartnerLeg1 as Carrier Partner (chặng 1, webhook)
    participant PollJob as Polling Job (tự phát hiện)
    participant Handler as Status Update Handler
    participant DB as Shipments + Status_Event Tables

    par 2 cập nhật gần như đồng thời từ 2 nguồn khác nhau
        PartnerLeg1->>Handler: Webhook "đã giao cho đối tác chặng 2" (external_event_id, event_time=T2)
    and
        PollJob->>Handler: Phát hiện "đang trung chuyển" (event_time=T1, T1 < T2)
    end

    Handler->>DB: Check external_event_id đã tồn tại trong shipment_status_event chưa
    alt external_event_id đã tồn tại, do partner retry
        DB-->>Handler: Duplicate, bỏ qua insert
        Handler-->>PartnerLeg1: Ack, no-op
    else Sự kiện mới
        Handler->>DB: BEGIN TRANSACTION
        Handler->>DB: INSERT INTO shipment_status_event (status, event_timestamp, external_event_id, source)
        Handler->>DB: SELECT MAX(event_timestamp) status hiện có cho shipment này
        alt event_timestamp của sự kiện mới lớn nhất
            Handler->>DB: UPDATE shipments SET status = new_status, status_updated_at = event_timestamp
        else event_timestamp nhỏ hơn trạng thái đang có, đến trễ
            Handler->>DB: Giữ nguyên cột status hiện tại, không ghi đè
        end
        Handler->>DB: COMMIT
        DB-->>Handler: Recorded
    end

    Note over DB: Dù T2 (webhook) đến trước T1 (polling) về mặt xử lý, status cuối cùng vẫn phản ánh đúng timestamp gốc lớn nhất, và cả 2 sự kiện đều được lưu đủ trong lịch sử
```
