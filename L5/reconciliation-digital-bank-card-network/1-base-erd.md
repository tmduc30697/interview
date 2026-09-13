# ERD — Base (trước khi có đối soát cuối ngày)

Đây là **base**: mô hình dữ liệu suy luận cho hệ thống ngân hàng số *trước khi* có flow đối soát. Đề bài giả định ngân hàng đã xử lý giao dịch thẻ qua đối tác (mạng thẻ) và ghi sổ cái nội bộ — nếu không có sẵn Account/Transaction/Ledger thì "đối soát ledger nội bộ với báo cáo đối tác" sẽ không có nghĩa. ERD chỉ dựng phần lõi phục vụ giao dịch thẻ + ghi sổ, không suy diễn thêm các module không liên quan (chuyển khoản nội địa, tiết kiệm, thẻ tín dụng...).

```mermaid
erDiagram
    CUSTOMER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ CARD : has
    ACCOUNT ||--o{ TRANSACTION : "posts to"
    TRANSACTION ||--|| LEDGER_ENTRY : records

    CUSTOMER {
        string customer_id PK
        string full_name
        string email
        string kyc_status
    }

    ACCOUNT {
        string account_id PK
        string customer_id FK
        string currency
        decimal balance
        string status
    }

    CARD {
        string card_id PK
        string account_id FK
        string card_network
        string card_number_masked
        string status
    }

    TRANSACTION {
        string transaction_id PK
        string card_id FK
        string account_id FK
        decimal amount
        string currency
        string merchant_name
        string status
        string partner_reference_id
        datetime created_at
    }

    LEDGER_ENTRY {
        string entry_id PK
        string transaction_id FK
        string account_id FK
        decimal amount
        string entry_type
        decimal balance_after
        datetime created_at
    }
```
