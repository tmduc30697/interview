# Sequence - Base: User tra cứu lịch sử giao dịch

Đây là **base**, flow "user tra cứu lịch sử giao dịch của tài khoản mình". Flow này được chọn vì đây chính là truy vấn mà đề bài muốn tối ưu bằng composite index `(account_id, created_at DESC)` - ở base, bảng chỉ có index đơn cột trên `account_id` nên DB phải tự sort kết quả, minh hoạ rõ vấn đề cần giải quyết.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile/Web App
    participant API as Transaction API
    participant DB as Postgres - transactions

    User->>App: Mở màn hình lịch sử giao dịch
    App->>API: GET /accounts/{account_id}/transactions?page=1
    API->>DB: SELECT * FROM transactions WHERE account_id = $1 ORDER BY created_at DESC LIMIT 20

    Note over DB: Planner dung idx_transactions_account_id (index don cot)
    Note over DB: de tim cac dong khop account_id
    Note over DB: nhung index nay KHONG luu theo thu tu created_at
    Note over DB: nen phai them buoc sort (filesort) truoc khi LIMIT 20

    DB-->>API: 20 dòng giao dịch gần nhất (sau khi sort)
    API-->>App: Danh sách giao dịch
    App-->>User: Hiển thị lịch sử giao dịch

    Note over DB: Bang cang lon (hang tram trieu dong),<br/>chi phi sort cho moi truy van cang tang,<br/>du chi lay 20 dong dau
```
