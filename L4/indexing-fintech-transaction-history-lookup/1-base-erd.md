# ERD - Base: bảng transactions cơ bản, chưa tối ưu index/partition

Đây là **base**, trạng thái dữ liệu ngay trước khi áp đề bài. Hệ thống ngân hàng số/ví điện tử đã có `ACCOUNT` và bảng `TRANSACTION` append-only lưu mọi giao dịch, nhưng bảng chưa được partition và chỉ có index tối thiểu (khóa chính trên `transaction_id`, index đơn cột trên `account_id`) - đây chính là điểm xuất phát mà đề bài (composite index, partial index, partition, read replica) cần cải thiện.

```mermaid
erDiagram
    ACCOUNT ||--o{ TRANSACTION : "owns"

    ACCOUNT {
        string account_id PK
        string customer_id
        string account_type
        string currency
        string status
        datetime created_at
    }

    TRANSACTION {
        string transaction_id PK
        string account_id FK
        datetime created_at
        string status
        string transaction_type
        decimal amount
        string currency
        string description
    }

    %% Index hien co (khong the hien truc tiep trong ERD):
    %% - PRIMARY KEY btree tren transaction_id
    %% - idx_transactions_account_id: index don cot tren account_id
    %% - KHONG co composite index (account_id, created_at)
    %% - KHONG co partial index theo status
    %% - Bang TRANSACTION la 1 heap duy nhat, chua partition theo thoi gian
```
