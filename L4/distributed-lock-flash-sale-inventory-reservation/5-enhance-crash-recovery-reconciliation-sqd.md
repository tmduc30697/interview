# Enhance sequence — Reconcile tồn kho bị kẹt khi process crash giữa chừng

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp process giữ lock crash ngay sau khi đã trừ tồn kho trong DB nhưng chưa kịp tạo `ORDER`/release lock. Đáp ứng yêu cầu 2: phát hiện trạng thái dở dang này khi lock hết hạn (đối chiếu số đã trừ với đơn hàng tồn tại tương ứng), để hoàn lại tồn kho bị kẹt thay vì mất vĩnh viễn.

```mermaid
sequenceDiagram
    actor Req as Request (process sẽ crash)
    participant App as Order Service (instance sẽ crash)
    participant Lock as Redis Lock (lock:sku:SKU-123)
    participant DB as Database
    participant Reconciler as Reconciliation Job

    Req->>App: Mua SKU-123
    App->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+1
    Lock-->>App: Giành lock thành công

    App->>DB: UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    App->>DB: INSERT INVENTORY_DEDUCTION (fencing_token=N+1, quantity_deducted=1, status=pending, order_id=null)

    Note over App: Process crash ngay tại đây, chưa kịp INSERT ORDER, chưa kịp DEL lock

    Note over Lock: Lock tự hết hạn sau 300ms vì không được release/renew

    loop Job chạy định kỳ (vd mỗi 5s)
        Reconciler->>DB: SELECT INVENTORY_DEDUCTION WHERE status=pending AND created_at < now() - lock_ttl_ms
        DB-->>Reconciler: Trả về deduction fencing_token=N+1, order_id=null, đã quá TTL mà vẫn pending
    end

    Reconciler->>DB: SELECT ORDER WHERE fencing_token_used=N+1
    DB-->>Reconciler: Không tìm thấy ORDER nào tương ứng, xác nhận đây là deduction dở dang thật sự

    Reconciler->>DB: UPDATE PRODUCT SET stock_quantity = stock_quantity + 1 (hoàn lại tồn kho bị kẹt)
    Reconciler->>DB: UPDATE INVENTORY_DEDUCTION SET status=refunded

    Note over Reconciler,DB: Tồn kho được hoàn lại đúng số lượng, sẵn sàng cho request tiếp theo, không bị mất vĩnh viễn do process crash giữa chừng
```
