# Sequence - Base: Retry sau timeout mạng (trừ tồn kho lần 2)

Đây là flow **base**: client bấm "Mua ngay", mạng chập chờn khiến response bị timeout dù request đã xử lý thành công ở backend, client không biết nên tự động retry với cùng thao tác. Vì không có idempotency key, backend coi request retry là 1 giao dịch mới hoàn toàn và trừ tồn kho lần nữa cho cùng 1 khách. Đây là tiền đề cho yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    participant U as Khách hàng
    participant API as Checkout API
    participant DB as Database

    U->>API: Bấm Mua ngay (lần 1)
    API->>DB: UPDATE stock_qty - 1, tạo ORDER #1 status=confirmed
    DB-->>API: Thành công
    API--xU: Response bị timeout trên đường mạng, khách không nhận được
    Note over U: Khách không biết request trước đã thành công hay chưa
    U->>API: Tự động retry, bấm Mua ngay lần nữa
    Note over API: Không có idempotency key để nhận diện đây là request lặp lại
    API->>DB: UPDATE stock_qty - 1 lần nữa, tạo ORDER #2 status=confirmed
    DB-->>API: Thành công
    API-->>U: Mua thành công
    Note over DB: Cùng 1 khách bị trừ tồn kho 2 lần cho 1 lần mua thực tế, tạo ra 2 ORDER trùng lặp
```
