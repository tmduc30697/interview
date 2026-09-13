# Enhance sequence — Lock per-SKU với TTL hợp lý và fail-fast cho request thua cuộc

Đây là **enhance** của flow `purchase-flash-sale` đã có ở base. So với base (đọc-trừ trực tiếp không khóa), enhance thêm distributed lock theo đúng granularity per-SKU (không lock toàn catalog, không lock chung theo warehouse) với TTL ngắn vừa đủ, và fail-fast cho request không giành được lock trong thời gian ngắn. Đáp ứng yêu cầu 1 (granularity per-SKU, TTL đủ ngắn để nhả nhanh nhưng đủ dài để hoàn tất trừ kho) và yêu cầu 4 (fail-fast, không xếp hàng chờ vô hạn khi có hàng nghìn request cạnh tranh).

```mermaid
sequenceDiagram
    actor Req1 as Request 1
    actor Req2 as Request 2
    actor ReqN as Request N (hàng nghìn request khác)
    participant App as Order Service
    participant Lock as Redis Lock (lock:sku:SKU-123, TTL 300ms)
    participant DB as Database

    Note over Req1,ReqN: Chỉ còn 1 đơn vị tồn kho SKU-123, hàng nghìn request cùng cạnh tranh

    Req1->>App: Mua SKU-123
    App->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+1
    Lock-->>App: Giành lock thành công

    par Các request khác cùng thử giành lock
        Req2->>App: Mua SKU-123
        App->>Lock: SET lock:sku:SKU-123 NX PX 300ms
        Lock-->>App: Thất bại, lock đang bị Req1 giữ
        App->>App: Chờ ngắn (vd 50ms), thử lại 1-2 lần
        App-->>Req2: Vẫn không giành được trong thời gian cho phép, trả "Hết hàng, vui lòng thử lại"

        ReqN->>App: Mua SKU-123
        App->>Lock: SET lock:sku:SKU-123 NX PX 300ms
        Lock-->>App: Thất bại
        App-->>ReqN: Fail-fast ngay, "Hết hàng, vui lòng thử lại" (không xếp hàng chờ)
    end

    App->>DB: (Req1) SELECT stock_quantity FROM PRODUCT WHERE sku=SKU-123
    DB-->>App: stock_quantity = 1
    App->>DB: UPDATE PRODUCT SET stock_quantity = 0, INSERT INVENTORY_DEDUCTION (fencing_token=N+1, status=pending)
    App->>DB: INSERT ORDER (fencing_token_used=N+1, status=confirmed)
    App->>DB: UPDATE INVENTORY_DEDUCTION SET status=confirmed, order_id=...
    App->>Lock: DEL lock:sku:SKU-123 (release ngay sau khi hoàn tất, chỉ ~vài chục ms)

    App-->>Req1: Mua thành công

    Note over App,Lock: Lock chỉ khóa đúng SKU-123, các SKU khác không bị ảnh hưởng, TTL 300ms đủ để hoàn tất trừ kho nhưng nhả rất nhanh
```
