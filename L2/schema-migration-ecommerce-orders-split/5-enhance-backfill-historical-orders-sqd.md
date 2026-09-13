# Sequence Diagram — Enhance: Backfill Historical Orders

Đây là **enhance**, flow hoàn toàn mới: job backfill đọc các order lịch sử (tạo trước khi có dual-write) theo cursor `order_id` tăng dần, batch nhỏ 500 order/lần, có checkpoint để resume nếu crash giữa chừng thay vì đọc lại từ đầu.

```mermaid
sequenceDiagram
    participant Job as Backfill Job
    participant Checkpoint as Backfill Checkpoint Store
    participant DB as Orders + Order_Items Tables

    Job->>Checkpoint: Read last_order_id_processed
    Checkpoint-->>Job: Resume cursor (or start from 0 if none)

    loop Until no more orders
        Job->>DB: SELECT next 500 orders WHERE order_id > cursor ORDER BY order_id
        DB-->>Job: Batch of orders with items_json

        loop for each order in batch
            Job->>Job: Parse items_json into item rows
            Job->>DB: INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        end

        Job->>Checkpoint: Update last_order_id_processed = max order_id in batch
        Checkpoint-->>Job: Checkpoint saved
    end

    Note over Job,Checkpoint: Nếu job crash giữa batch, lần chạy sau resume đúng từ checkpoint đã lưu, không đọc lại từ đầu
    Job-->>Job: Backfill completed
```
