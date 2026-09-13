# Sequence - Enhance: Retry sau timeout mạng (idempotent, không trừ tồn kho lần 2)

Đây là flow **enhance** của `retry-after-timeout` (so với base). Khác biệt so với base: client gửi kèm `idempotency_key` cố định cho lần bấm mua đó (kể cả khi retry), backend kiểm tra key này đã tồn tại `ORDER` hay chưa trước khi chạy transaction trừ tồn kho — nếu đã tồn tại, trả thẳng lại kết quả của lần xử lý đầu tiên, không chạy lại transaction. Đáp ứng **yêu cầu 4** của đề bài.

```mermaid
sequenceDiagram
    participant U as Khách hàng
    participant API as Checkout API
    participant DB as Database

    U->>API: Bấm Mua ngay (lần 1, idempotency_key=abc123)
    API->>DB: Kiểm tra ORDER với idempotency_key=abc123 (chưa có)
    API->>DB: UPDATE stock_qty - 1 WHERE stock_qty > 0
    DB-->>API: affected_rows=1
    API->>DB: Tạo ORDER(idempotency_key=abc123, status=confirmed)
    DB-->>API: Thành công
    API--xU: Response bị timeout trên đường mạng, khách không nhận được
    Note over U: Khách không biết request trước đã thành công hay chưa
    U->>API: Tự động retry, cùng idempotency_key=abc123
    API->>DB: Kiểm tra ORDER với idempotency_key=abc123
    DB-->>API: Đã tồn tại ORDER, status=confirmed
    Note over API: Không chạy lại transaction trừ tồn kho, chỉ trả lại đúng kết quả cũ
    API-->>U: Mua thành công (kết quả của lần xử lý đầu tiên)
    Note over DB: Tồn kho chỉ bị trừ đúng 1 lần cho 1 lần mua thực tế, dù client retry bao nhiêu lần
```
