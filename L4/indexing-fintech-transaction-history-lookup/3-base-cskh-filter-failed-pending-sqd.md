# Sequence - Base: CSKH lọc giao dịch failed/pending

Đây là **base**, flow "CSKH tra cứu toàn bộ giao dịch failed/pending để xử lý sự cố". Flow này được chọn vì đề bài đề xuất partial index cho đúng use case này - ở base, bảng chưa có bất kỳ index nào trên cột `status`, nên minh hoạ vấn đề: dù số dòng failed/pending chỉ là phần rất nhỏ, DB vẫn phải quét toàn bộ bảng.

```mermaid
sequenceDiagram
    actor CSKH as CSKH Agent
    participant Tool as Internal Ops Tool
    participant DB as Postgres - transactions

    CSKH->>Tool: Xem danh sách giao dịch failed/pending
    Tool->>DB: SELECT * FROM transactions WHERE status IN ('failed','pending') ORDER BY created_at DESC LIMIT 100

    Note over DB: Khong co index nao tren cot status
    Note over DB: Planner phai Seq Scan toan bo bang transactions
    Note over DB: (hang tram trieu dong) de tim ra so it dong khop dieu kien

    DB-->>Tool: 100 dòng failed/pending (sau khi quét toàn bảng)
    Tool-->>CSKH: Danh sách giao dịch cần xử lý

    Note over DB: Query nay chay thuong xuyen nhung rat ton I/O,<br/>vi phai doc ca cac dong status = success<br/>chiem da so bang de loc ra phan nho con lai
```
