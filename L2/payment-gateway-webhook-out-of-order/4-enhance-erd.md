# Enhance ERD — Thêm log webhook idempotent theo thứ tự thật và đối soát hàng ngày

Đây là **enhance**, mô hình dữ liệu sau khi áp cơ chế xử lý webhook idempotent + đúng thứ tự thật lên base. So với base, thêm mới `WEBHOOK_EVENT` — lưu lại từng webhook nhận được kèm `gateway_event_id` unique (chống xử lý trùng) và `gateway_sequence_number`/`gateway_event_at` (thứ tự thật từ phía cổng thanh toán, dùng để quyết định trạng thái cuối thay vì thời điểm app nhận). `PAYMENT_TRANSACTION` thêm `last_applied_sequence_number` để biết event nào đã áp dụng gần nhất, tránh bị 1 event cũ hơn đến muộn ghi đè ngược. `ORDER` thêm trạng thái `refund_pending` cho luồng hoàn tiền khi hủy đơn đụng webhook success. Thêm mới `RECONCILIATION_DISCREPANCY` cho job đối soát hàng ngày.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ PAYMENT_TRANSACTION : "paid via"
    ORDER ||--o{ WEBHOOK_EVENT : "receives"
    PAYMENT_TRANSACTION ||--o{ WEBHOOK_EVENT : "derived from"

    CUSTOMER {
        string id PK
        string name
    }
    ORDER {
        string id PK
        string customer_id FK
        decimal amount
        string status "pending | processing | paid | cancelled | refund_pending | refunded"
    }
    PAYMENT_TRANSACTION {
        string id PK
        string order_id FK
        string gateway_transaction_id
        string status "pending | success | refunded | failed"
        int last_applied_sequence_number "sequence number của webhook mới nhất đã áp dụng"
        datetime updated_at
    }
    WEBHOOK_EVENT {
        string id PK
        string order_id FK
        string payment_transaction_id FK
        string gateway_event_id UK "idempotency key, chống xử lý trùng"
        string event_type "payment_success | payment_refunded | payment_failed"
        int gateway_sequence_number "thứ tự thật do cổng thanh toán phát ra"
        datetime gateway_event_at "timestamp thật của sự kiện tại cổng thanh toán"
        datetime received_at "thời điểm app nhận được, chỉ để log, không dùng để quyết định trạng thái"
        boolean applied "đã áp dụng vào PAYMENT_TRANSACTION hay bị bỏ qua vì đến muộn hơn event đã áp dụng"
    }
    RECONCILIATION_DISCREPANCY {
        string id PK
        string gateway_transaction_id
        string order_id FK "nullable, có thể null nếu giao dịch chỉ tồn tại ở phía cổng thanh toán"
        string discrepancy_type "missing_internally | missing_at_gateway"
        date report_date
        string status "open | resolved"
        datetime detected_at
    }
```
