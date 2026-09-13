# ERD — Base (trước khi có đối soát payout seller)

Đây là **base**: mô hình dữ liệu suy luận cho marketplace đa seller *trước khi* có flow đối soát payout. Đề bài giả định sàn đã thu tiền từ người mua, đã tính hoa hồng theo cấu hình phí, và đã có cơ chế tính/chuyển tiền định kỳ cho seller — nếu không có sẵn `ORDER`, `COMMISSION_CONFIG` và `PAYOUT` thì "đối soát số tiền tính toán với số tiền thực chuyển" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ payout, không suy diễn thêm các module không liên quan (catalog, review...).

```mermaid
erDiagram
    SELLER ||--o{ ORDER : fulfills
    COMMISSION_CONFIG ||--o{ ORDER : "applies to"
    SELLER ||--o{ PAYOUT : receives
    PAYOUT ||--o{ ORDER : aggregates
    PAYOUT ||--o| PAYOUT_TRANSFER : "sent via"

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
        string status
        datetime completed_at
    }

    COMMISSION_CONFIG {
        string config_id PK
        decimal commission_rate
        datetime effective_from
    }

    PAYOUT {
        string payout_id PK
        string seller_id FK
        date period_start
        date period_end
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
    }
```
