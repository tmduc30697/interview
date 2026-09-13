# Sequence - Enhance: User tra cứu lịch sử giao dịch

Đây là **enhance** của flow "user tra cứu lịch sử giao dịch" đã có ở base (file `2-base-user-transaction-history-lookup-sqd.md`). So với base, DB không còn phải filesort: bảng đã range-partition theo `created_at` nên planner prune bớt partition không liên quan, rồi dùng local composite index `(account_id, created_at DESC)` của từng partition còn lại để lấy dữ liệu đã sẵn đúng thứ tự.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile/Web App
    participant API as Transaction API
    participant DB as Postgres - transactions (partitioned)

    User->>App: Mở màn hình lịch sử giao dịch
    App->>API: GET /accounts/{account_id}/transactions?page=1
    API->>DB: SELECT * FROM transactions WHERE account_id = $1 ORDER BY created_at DESC LIMIT 20

    Note over DB: Partition pruning: chi can xet cac partition<br/>thang/nam gan nhat (nhung partition cu hon<br/>bi loai truoc khi quet)

    loop Với mỗi partition còn lại (mới nhất trước)
        Note over DB: Dung local index (account_id, created_at DESC)<br/>cua partition do - vua loc account_id<br/>vua tra ve dung thu tu created_at DESC
        DB-->>DB: Index scan tra ve dong da sort san, KHONG can filesort
    end

    DB-->>API: 20 dòng giao dịch gần nhất (gộp từ các partition, không filesort)
    API-->>App: Danh sách giao dịch
    App-->>User: Hiển thị lịch sử giao dịch

    Note over DB: So voi base: khong con buoc sort rieng,<br/>va moi partition co index nho gon nen<br/>toc do on dinh du bang tong the ngay cang lon
```
