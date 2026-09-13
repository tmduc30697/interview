# ERD - Enhance: transactions range-partition + partial index + read replica

Đây là **enhance**, đúng nguyên văn nội dung đề bài. So với base, bảng `TRANSACTION` logic nay được range-partition theo tháng/năm dựa trên `created_at`, mỗi partition mang bộ index riêng nhỏ gọn (composite `(account_id, created_at DESC)` cho user, partial index theo `status` cho CSKH), và có thêm `READ_REPLICA` tách riêng cho compliance/audit để không cạnh tranh tài nguyên với luồng giao dịch thời gian thực.

```mermaid
erDiagram
    ACCOUNT ||--o{ TRANSACTION : "owns"
    TRANSACTION ||--|{ TRANSACTION_PARTITION : "partitioned by month/year (created_at)"
    TRANSACTION ||--o{ READ_REPLICA : "streaming replication"

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
        datetime created_at "partition key"
        string status
        string transaction_type
        decimal amount
        string currency
        string description
    }

    TRANSACTION_PARTITION {
        string partition_name PK
        date range_start
        date range_end
        string local_composite_index "btree (account_id, created_at DESC)"
        string local_partial_index "btree (status) WHERE status IN (pending,failed)"
    }

    READ_REPLICA {
        string replica_id PK
        string source_table FK
        string purpose "compliance/audit full-scan queries"
        string replication_lag
    }

    %% Quyet dinh thiet ke tu de bai:
    %% - KHONG them status/transaction_type vao composite (account_id, created_at)
    %%   de giu index chinh cho user gon nhe, CSKH loc status/type dung index rieng
    %%   hoac chap nhan index intersection cho truong hop it gap
    %% - Partial index tren status chi chua rows pending/failed (~ ty le nho)
    %% - Moi partition co bo index rieng, nho gon, khong phinh to theo thoi gian
    %% - Insert theo thoi gian van co the tao hot page trong 1 partition,
    %%   can them chien luoc giam contention (vd shard/hash theo account_id)
```

