# ERD — Enhance (sau khi tách order_items)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu của đề bài — dual-write `order_items` quan hệ song song với `items_json` cũ, checkpoint cho job backfill, và báo cáo đối soát. So với base, `ORDER` giữ nguyên cột `items_json` (chưa drop, vẫn đang dual-write) và có thêm `idempotency_key` để chống double-submit; quan hệ `ORDER` — `PRODUCT` giờ đi qua `ORDER_ITEM` thật sự là FK thay vì chỉ nằm lỏng lẻo trong JSON.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ ORDER_ITEM : "contains (new, dual-written)"
    ORDER_ITEM }o--|| PRODUCT : references
    RECONCILIATION_REPORT }o--|| ORDER : samples

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        string items_json
        string idempotency_key
        decimal total_amount
        string status
        datetime created_at
    }

    ORDER_ITEM {
        string order_item_id PK
        string order_id FK
        string product_id FK
        int quantity
        decimal unit_price
    }

    PRODUCT {
        string product_id PK
        string name
        decimal price
        int stock_quantity
    }

    BACKFILL_CHECKPOINT {
        string checkpoint_id PK
        string last_order_id_processed
        int batch_size
        string status
        datetime updated_at
    }

    RECONCILIATION_REPORT {
        string report_id PK
        date run_date
        string sample_order_id FK
        decimal json_total
        decimal relational_total
        boolean mismatch_found
    }
```
