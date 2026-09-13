# Enhance sequence — Idempotency key chống trừ kho 2 lần khi client retry

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp client retry do timeout network. Đáp ứng yêu cầu 5: lock chỉ đảm bảo tuần tự xử lý chứ không đảm bảo request không bị gửi lặp, nên cần idempotency key gắn với request mua hàng, độc lập hoàn toàn với cơ chế lock, để đảm bảo retry không gây trừ kho 2 lần cho cùng 1 yêu cầu.

```mermaid
sequenceDiagram
    actor Client
    participant App as Order Service
    participant DB as Database (IDEMPOTENCY_RECORD)
    participant Lock as Redis Lock (lock:sku:SKU-123)

    Client->>App: Mua SKU-123 (idempotency_key=abc-123)
    App->>DB: SELECT IDEMPOTENCY_RECORD WHERE idempotency_key=abc-123
    DB-->>App: Không tìm thấy, đây là lần đầu

    App->>DB: INSERT IDEMPOTENCY_RECORD (idempotency_key=abc-123, result_status=processing)
    App->>Lock: SET lock:sku:SKU-123 NX PX 300ms
    Lock-->>App: Giành lock thành công

    App->>DB: Trừ kho, INSERT ORDER
    App->>Lock: DEL lock:sku:SKU-123

    App->>DB: UPDATE IDEMPOTENCY_RECORD SET result_status=success, order_id=...

    Note over Client,App: Response bị mất do network timeout, client không nhận được kết quả

    Client->>App: Retry, gửi lại request mua SKU-123 (cùng idempotency_key=abc-123)
    App->>DB: SELECT IDEMPOTENCY_RECORD WHERE idempotency_key=abc-123
    DB-->>App: Tìm thấy, result_status=success, order_id=...

    Note over App: Không lặp lại bước giành lock, không trừ kho lần nữa
    App-->>Client: Trả thẳng kết quả order đã có, "Mua thành công" (idempotent, không trừ kho lần 2)

    Note over App,DB: Idempotency check nằm hoàn toàn trước và độc lập với bước giành lock, nên bảo vệ được cả trường hợp lock đã release xong trước khi client kịp retry
```
