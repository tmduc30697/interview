# ERD — Enhance (sau khi có đối soát cuốc xe/tài xế/hoa hồng)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng 5 yêu cầu của đề bài — cuốc bị hủy giữa chừng có phí một phần, giá cước điều chỉnh sau khi cuốc kết thúc, thanh toán tiền mặt theo dõi riêng, điều chỉnh phát sinh từ tranh chấp giá, và đối soát ở cả mức cuốc lẫn mức đợt thanh toán. So với base, `WALLET_ENTRY` nay còn được tạo bởi `FARE_ADJUSTMENT` và `TRIP_CANCELLATION` (không chỉ bởi cuốc hoàn tất bình thường), và `TRIP` có thêm cờ `payment_method` để phân biệt tiền mặt.

```mermaid
erDiagram
    RIDER ||--o{ TRIP : requests
    DRIVER ||--o{ TRIP : completes
    TRIP ||--|| FARE : has
    COMMISSION_CONFIG ||--o{ TRIP : "applies to"
    DRIVER ||--|| DRIVER_WALLET : has
    TRIP ||--o{ WALLET_ENTRY : "credits/debits"
    DRIVER_WALLET ||--o{ WALLET_ENTRY : accumulates

    TRIP |o--o| TRIP_CANCELLATION : "may end in"
    FARE ||--o{ FARE_ADJUSTMENT : "may have"
    FARE_ADJUSTMENT |o--o| WALLET_ENTRY : generates

    DRIVER_WALLET ||--o{ PAYOUT_BATCH : "settled via"
    PAYOUT_BATCH ||--o{ WALLET_ENTRY : aggregates

    RECONCILIATION_RUN ||--o{ RECONCILIATION_DISCREPANCY : yields
    RECONCILIATION_DISCREPANCY |o--o| TRIP : "relates to"
    RECONCILIATION_DISCREPANCY |o--o| PAYOUT_BATCH : "relates to"

    RIDER {
        string rider_id PK
        string full_name
        string phone
    }

    DRIVER {
        string driver_id PK
        string full_name
        string phone
        string status
    }

    TRIP {
        string trip_id PK
        string rider_id FK
        string driver_id FK
        string status
        string payment_method
        datetime requested_at
        datetime completed_at
    }

    FARE {
        string fare_id PK
        string trip_id FK
        decimal estimated_amount
        decimal final_amount
        string currency
    }

    FARE_ADJUSTMENT {
        string adjustment_id PK
        string fare_id FK
        decimal amount_diff
        string reason
        string approved_by
        datetime created_at
    }

    TRIP_CANCELLATION {
        string cancellation_id PK
        string trip_id FK
        string cancelled_by
        decimal cancellation_fee
        string reason
        datetime cancelled_at
    }

    COMMISSION_CONFIG {
        string config_id PK
        decimal commission_rate
        datetime effective_from
    }

    DRIVER_WALLET {
        string wallet_id PK
        string driver_id FK
        decimal balance
    }

    WALLET_ENTRY {
        string entry_id PK
        string wallet_id FK
        string trip_id FK
        string adjustment_id FK
        decimal amount
        string entry_type
        datetime created_at
    }

    PAYOUT_BATCH {
        string batch_id PK
        string driver_id FK
        date period_start
        date period_end
        decimal total_amount
        string status
    }

    RECONCILIATION_RUN {
        string run_id PK
        date period_start
        date period_end
        string status
    }

    RECONCILIATION_DISCREPANCY {
        string discrepancy_id PK
        string run_id FK
        string trip_id FK
        string batch_id FK
        string level
        decimal amount_diff
        string status
    }
```
