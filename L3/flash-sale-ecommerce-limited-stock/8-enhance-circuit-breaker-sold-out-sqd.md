# Sequence - Enhance: Circuit breaker chặn request khi đã hết hàng

Đây là flow **enhance hoàn toàn mới**, mô tả cơ chế circuit breaker/giảm tải khi tồn kho đã về 0. Khi request cuối cùng trừ tồn kho về 0 (hoặc thất bại vì `affected_rows=0`), hệ thống lập tức bật cờ "hết hàng" ở tầng cache/API gateway, mọi request đến sau bị chặn ngay tại đó, không để lọt xuống DB gây tải vô ích. Đáp ứng **yêu cầu 5** của đề bài.

```mermaid
sequenceDiagram
    participant API as Checkout API
    participant DB as Database
    participant Cache as Cache (CIRCUIT_BREAKER_STATE)
    participant GW as API Gateway
    participant LateUser as Khách đến sau khi đã hết hàng

    API->>DB: UPDATE stock_qty - 1 WHERE stock_qty > 0 (request cuối cùng)
    DB-->>API: affected_rows=1, stock_qty vừa về 0
    API->>Cache: Set CIRCUIT_BREAKER_STATE(product_id=X, state=open, tripped_at=now)
    Note over Cache: Breaker chuyển open ngay khi phát hiện stock_qty=0, không chờ request tiếp theo mới phát hiện

    LateUser->>GW: Bấm Mua ngay (đến sau khi đã hết hàng)
    GW->>Cache: Kiểm tra CIRCUIT_BREAKER_STATE(product_id=X)
    Cache-->>GW: state=open
    GW-->>LateUser: Hết hàng (phản hồi ngay tại gateway, không gọi xuống Checkout API hay DB)
    Note over DB: DB hoàn toàn không nhận thêm request nào cho sản phẩm X sau khi đã hết hàng, tránh tải vô ích lúc cao điểm
```
