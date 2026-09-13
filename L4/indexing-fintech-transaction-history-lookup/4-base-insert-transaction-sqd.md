# Sequence - Base: Insert giao dịch mới (append-only)

Đây là **base**, flow "hệ thống thanh toán insert giao dịch mới vào bảng transactions". Flow này được chọn vì đề bài nhắc tới 2 vấn đề trực tiếp liên quan tới insert: (1) B-tree của bảng đơn (chưa partition) ngày càng phình to làm chậm ghi và bảo trì index, (2) giờ cao điểm insert đồng thời vào cột tăng dần (created_at/id) gây hot page ở cuối B-tree.

```mermaid
sequenceDiagram
    participant Pay as Payment Service
    participant DB as Postgres - transactions

    Pay->>DB: INSERT INTO transactions (transaction_id, account_id, created_at, status, ...)

    Note over DB: Chi co 1 bang transactions duy nhat (chua partition)
    Note over DB: Moi insert phai cap nhat CA HAI index tren toan bang:
    Note over DB: PRIMARY KEY (transaction_id) va idx_transactions_account_id

    Note over DB: transaction_id/created_at tang dan deu,<br/>nen cac insert moi lien tuc roi vao cung<br/>1 trang (leaf page) cuoi cung cua B-tree

    Note over DB: Gio cao diem: nhieu transaction insert dong thoi<br/>tranh chap lock/ghi tren cung hot page do

    DB-->>Pay: Insert thành công (nhưng chậm dần khi bảng lớn lên)

    Note over DB: Kich thuoc bang va index chi tang, khong co<br/>co che nao giu index nho gon theo thoi gian
```
