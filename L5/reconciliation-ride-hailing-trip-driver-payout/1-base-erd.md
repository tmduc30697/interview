# ERD — Base (trước khi có đối soát cuốc xe/tài xế/hoa hồng)

Đây là **base**: mô hình dữ liệu suy luận cho ứng dụng gọi xe *trước khi* có flow đối soát ba chiều (khách trả - tài xế nhận - hoa hồng nền tảng). Đề bài giả định hệ thống đã có cuốc xe với giá cước, đã tính hoa hồng và đã chia tiền cho tài xế qua ví — nếu không có sẵn `TRIP`, `FARE` và `DRIVER_WALLET` thì "đối soát ba khoản khớp đúng cho từng cuốc" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ cuốc xe + chia tiền, không suy diễn thêm các module không liên quan (khuyến mãi, đánh giá tài xế...).

```mermaid
erDiagram
    RIDER ||--o{ TRIP : requests
    DRIVER ||--o{ TRIP : completes
    TRIP ||--|| FARE : has
    COMMISSION_CONFIG ||--o{ TRIP : "applies to"
    DRIVER ||--|| DRIVER_WALLET : has
    TRIP ||--o| WALLET_ENTRY : "credits/debits"
    DRIVER_WALLET ||--o{ WALLET_ENTRY : accumulates

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
        decimal amount
        string entry_type
        datetime created_at
    }
```
