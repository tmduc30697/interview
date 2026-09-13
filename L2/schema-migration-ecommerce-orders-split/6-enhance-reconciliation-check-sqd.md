# Sequence Diagram — Enhance: Reconciliation Check

Đây là **enhance**, flow hoàn toàn mới: job đối soát hằng ngày so sánh số lượng/tổng tiền giữa cột `items_json` cũ và bảng `order_items` mới cho một mẫu order ngẫu nhiên, cảnh báo nếu lệch — điều kiện bắt buộc trước khi được phép drop cột JSON cũ.

```mermaid
sequenceDiagram
    participant Job as Reconciliation Job
    participant DB as Orders + Order_Items Tables
    participant Alert as Alert Service

    Job->>DB: Pick random sample of orders for today
    DB-->>Job: Sample order list

    loop for each sampled order
        Job->>DB: Read items_json for order
        Job->>DB: Read order_items rows for order
        Job->>Job: Compute total/count from items_json
        Job->>Job: Compute total/count from order_items
        alt Totals match
            Job->>Job: Record report, mismatch_found = false
        else Totals differ
            Job->>Job: Record report, mismatch_found = true
            Job->>Alert: Trigger alert (order_id, json_total, relational_total)
        end
    end

    Job-->>Job: Reconciliation run completed
    Note over Job: Chỉ khi dual-write chạy ổn định, không lệch trong thời gian tối thiểu quy định và 100% đọc đã chuyển sang order_items thì mới được drop items_json
```
