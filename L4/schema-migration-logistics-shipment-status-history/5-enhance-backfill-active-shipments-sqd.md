# Sequence Diagram — Enhance: Backfill Active Shipments

Đây là **enhance**, flow hoàn toàn mới: job backfill tạo `SHIPMENT_STATUS_EVENT` khởi tạo cho các vận đơn đang active từ trạng thái hiện có, dùng khóa ngắn trên từng vận đơn để không bị real-time update đến giữa chừng ghi đè mất.

```mermaid
sequenceDiagram
    participant Job as Backfill Job
    participant Lock as Backfill Lock Service
    participant Handler as Status Update Handler
    participant DB as Shipments + Status_Event Tables

    loop for each active shipment chưa backfill
        Job->>Lock: Acquire short lock cho shipment_id
        alt Lock đang bị real-time update giữ
            Lock-->>Job: Lock busy, retry sau
        else Lock acquired
            Job->>DB: Đọc status hiện tại của shipment
            Job->>DB: INSERT INTO shipment_status_event (status hiện tại, event_timestamp = status_updated_at, source = backfill)
            Job->>DB: Kiểm tra lại status có đổi trong lúc backfill không
            alt Status không đổi trong lúc backfill
                Job->>Lock: Release lock
            else Real-time update đến giữa chừng, status đã đổi
                Note over Handler,DB: Update thật đã tự ghi event mới qua flow update-shipment-status, backfill event không ghi đè lên nó
                Job->>Lock: Release lock
            end
        end
    end

    Job-->>Job: Backfill completed for all active shipments
```
