# ERD — Enhance (sau khi có đối soát payout seller)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 4 yêu cầu của đề bài — cutoff nhất quán theo kỳ, xử lý hoàn tiền phát sinh sau payout (clawback), phân biệt "đã lên lệnh chuyển" với "seller thực nhận", và hold kỳ payout tiếp theo khi phát hiện sai lệch. So với base, `ORDER` nay lưu thêm `commission_rate_snapshot` chốt tại thời điểm hoàn tất (không tính lại theo cấu hình hiện tại), và `PAYOUT_TRANSFER` có thêm trạng thái xác nhận rõ ràng.

```mermaid
erDiagram
    SELLER ||--o{ ORDER : fulfills
    COMMISSION_CONFIG ||--o{ ORDER : "applies to"
    SELLER ||--o{ PAYOUT : receives
    PAYOUT ||--o{ ORDER : aggregates
    PAYOUT ||--o| PAYOUT_TRANSFER : "sent via"

    PAYOUT_PERIOD ||--o{ PAYOUT : defines
    ORDER ||--o{ REFUND_EVENT : "may have"
    REFUND_EVENT |o--o| PAYOUT_CLAWBACK : "triggers"
    PAYOUT_CLAWBACK |o--o| PAYOUT : "deducted in next"

    RECONCILIATION_RUN ||--o{ RECONCILIATION_DISCREPANCY : yields
    RECONCILIATION_DISCREPANCY |o--o| PAYOUT : "relates to"
    RECONCILIATION_DISCREPANCY |o--o| PAYOUT_HOLD : "may trigger"
    PAYOUT_HOLD |o--o| PAYOUT : "holds next"

    SELLER {
        string seller_id PK
        string business_name
        string bank_account_info
        string status
    }

    ORDER {
        string order_id PK
        string seller_id FK
        decimal order_amount
        decimal commission_amount
        decimal commission_rate_snapshot
        string status
        datetime completed_at
        string payout_id FK
    }

    COMMISSION_CONFIG {
        string config_id PK
        decimal commission_rate
        datetime effective_from
    }

    PAYOUT_PERIOD {
        string period_id PK
        date period_start
        date period_end
        datetime cutoff_at
        string status
    }

    PAYOUT {
        string payout_id PK
        string seller_id FK
        string period_id FK
        decimal total_amount
        string status
    }

    PAYOUT_TRANSFER {
        string transfer_id PK
        string payout_id FK
        decimal amount
        string gateway_reference_code
        string status
        datetime initiated_at
        datetime confirmed_at
    }

    REFUND_EVENT {
        string refund_event_id PK
        string order_id FK
        decimal amount
        datetime occurred_at
    }

    PAYOUT_CLAWBACK {
        string clawback_id PK
        string refund_event_id FK
        string applied_to_payout_id FK
        decimal amount
        string status
    }

    RECONCILIATION_RUN {
        string run_id PK
        string period_id FK
        datetime started_at
        datetime completed_at
        string status
    }

    RECONCILIATION_DISCREPANCY {
        string discrepancy_id PK
        string run_id FK
        string payout_id FK
        decimal amount_diff
        string type
        string status
    }

    PAYOUT_HOLD {
        string hold_id PK
        string seller_id FK
        string discrepancy_id FK
        string held_payout_id FK
        string reason
        string status
        datetime created_at
        datetime released_at
    }
```
