# Sequence - Enhance: Insert giao dịch mới (partition + giảm hot page contention)

Đây là **enhance** của flow "insert giao dịch mới" đã có ở base (file `4-base-insert-transaction-sqd.md`). So với base, insert nay chỉ cần cập nhật index của đúng 1 partition (nhỏ gọn hơn nhiều so với index toàn bảng cũ), và có thêm bước phân tán ghi theo `account_id` để giảm tình trạng nhiều transaction cùng tranh chấp một hot page ở cuối B-tree trong giờ cao điểm.

```mermaid
sequenceDiagram
    participant Pay as Payment Service
    participant Router as Partition/Write Router
    participant DB as Postgres - transactions (partitioned)

    Pay->>Router: INSERT transaction (account_id, created_at, status, ...)
    Router->>Router: Xác định partition đích theo created_at (tháng/năm hiện tại)

    Note over Router: Ket hop them buoc phan tan ghi theo hash(account_id)<br/>de cac insert cung thoi diem khong don het<br/>vao 1 vung leaf page duy nhat cua partition

    Router->>DB: INSERT vào đúng partition (vd transactions_y2026m09)

    Note over DB: Chi 2 index CUC BO cua partition nay can cap nhat:<br/>composite (account_id, created_at DESC) va partial index status
    Note over DB: Index nho hon nhieu so voi index toan bang o base,<br/>nen chi phi maintain B-tree thap hon

    DB-->>Router: Insert thành công
    Router-->>Pay: Ack

    Note over DB: So voi base: (1) index can cap nhat nho gon<br/>hon vi chi thuoc 1 partition, (2) phan tan ghi<br/>theo account_id giam tranh chap hot page<br/>so voi ghi tuan tu thuan theo thoi gian
```
