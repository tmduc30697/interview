# ERD — Enhance (sau khi có đối soát cổng thanh toán)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — chạy job đối soát khớp theo mã tham chiếu, phân loại rõ 4 loại sai lệch, xử lý refund/chargeback phát sinh sau khi đã khớp, đảm bảo job chạy lại (re-run) không tạo trùng lặp, và hàng đợi xử lý thủ công có theo dõi trạng thái cho sai lệch không tự giải quyết được. So với base, `ORDER` và `PAYMENT_TRANSACTION` không đổi cấu trúc, chỉ được tham chiếu thêm bởi các entity đối soát mới.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o| PAYMENT_TRANSACTION : "paid via"

    RECONCILIATION_RUN ||--o{ RECONCILIATION_RESULT : produces
    RECONCILIATION_RESULT |o--o| ORDER : "matches (optional)"
    RECONCILIATION_RESULT |o--o| PAYMENT_TRANSACTION : "matches (optional)"
    RECONCILIATION_RESULT ||--o| DISCREPANCY : "may yield"

    DISCREPANCY |o--o| MANUAL_REVIEW_ITEM : "escalated to"
    PAYMENT_TRANSACTION ||--o{ REFUND_EVENT : "may have"
    REFUND_EVENT |o--o| DISCREPANCY : "re-opens"

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
    }

    ORDER {
        string order_id PK
        string customer_id FK
        decimal total_amount
        string currency
        string payment_reference_code
        string payment_status
        datetime created_at
    }

    PAYMENT_TRANSACTION {
        string transaction_id PK
        string gateway_reference_code
        string order_id FK
        decimal amount
        string currency
        string status
        datetime received_at
    }

    RECONCILIATION_RUN {
        string run_id PK
        date period_start
        date period_end
        datetime started_at
        datetime completed_at
        string status
    }

    RECONCILIATION_RESULT {
        string result_id PK
        string run_id FK
        string order_id FK
        string transaction_id FK
        string match_status
        datetime created_at
    }

    DISCREPANCY {
        string discrepancy_id PK
        string result_id FK
        string type
        decimal amount_diff
        string status
        datetime created_at
        datetime resolved_at
    }

    MANUAL_REVIEW_ITEM {
        string review_id PK
        string discrepancy_id FK
        string assigned_to
        string status
        datetime created_at
        datetime resolved_at
    }

    REFUND_EVENT {
        string refund_event_id PK
        string transaction_id FK
        string type
        decimal amount
        datetime occurred_at
        datetime processed_at
    }
```
