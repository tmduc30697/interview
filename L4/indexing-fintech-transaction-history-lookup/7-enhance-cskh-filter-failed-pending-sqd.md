# Sequence - Enhance: CSKH lọc giao dịch failed/pending

Đây là **enhance** của flow "CSKH lọc giao dịch failed/pending" đã có ở base (file `3-base-cskh-filter-failed-pending-sqd.md`). So với base, mỗi partition nay có sẵn partial index chỉ chứa các dòng `status IN (pending, failed)`, nên thay vì Seq Scan toàn bảng, DB chỉ cần Bitmap Index Scan trên partial index rất nhỏ của từng partition.

```mermaid
sequenceDiagram
    actor CSKH as CSKH Agent
    participant Tool as Internal Ops Tool
    participant DB as Postgres - transactions (partitioned)

    CSKH->>Tool: Xem danh sách giao dịch failed/pending
    Tool->>DB: SELECT * FROM transactions WHERE status IN ('failed','pending') ORDER BY created_at DESC LIMIT 100

    loop Với mỗi partition
        Note over DB: Dung local partial index<br/>btree (status) WHERE status IN (pending,failed)
        Note over DB: Index nay chi chua vai % so dong cua partition,<br/>nen Bitmap Index Scan rat nhanh
    end

    DB-->>Tool: 100 dòng failed/pending (không cần quét dòng success)
    Tool-->>CSKH: Danh sách giao dịch cần xử lý

    Note over DB: So voi base: khong con Seq Scan toan bang,<br/>chi phi query gan nhu ty le voi so dong<br/>pending/failed thuc te, khong phu thuoc<br/>tong kich thuoc bang
```
