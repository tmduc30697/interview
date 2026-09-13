# Enhance sequence — Update product (invalidate theo event + TTL ngắn dự phòng + đo staleness)

Đây là **enhance**, cùng flow "Update product" đã có ở base nhưng nay thay đổi theo yêu cầu 1, 2 và 5 của đề bài: cập nhật giá/tồn kho phát ngay `CACHE_INVALIDATION_EVENT` để invalidate trong vài giây, kết hợp TTL ngắn dự phòng phòng khi event bị lỡ, và đo staleness thực tế.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminApp as Admin Panel
    participant DB as PRODUCT store
    participant EventBus as Cache Invalidation Event
    participant Cache as PRODUCT_CACHE_ENTRY store
    participant Metric as STALENESS_METRIC store

    Admin->>AdminApp: Đổi giá hoặc tồn kho sản phẩm
    AdminApp->>DB: Cập nhật PRODUCT (price/stock, version += 1, updated_at)
    DB-->>AdminApp: Cập nhật thành công (version mới)
    AdminApp->>EventBus: Publish CACHE_INVALIDATION_EVENT (product_id, changed_fields, source_version=version mới)
    EventBus->>Cache: Xoá/đánh dấu invalid PRODUCT_CACHE_ENTRY(product_id) ngay lập tức
    Cache-->>EventBus: Đã invalidate (trong vài giây kể từ lúc đổi)
    EventBus->>Metric: Ghi STALENESS_METRIC (changed_at=updated_at, cache_consistent_at=now)
    AdminApp-->>Admin: "Cập nhật thành công"

    Note over Cache: TTL ngắn (30-60s) vẫn được giữ làm dự phòng — nếu vì lý do gì đó event bị lỡ (mất kết nối, consumer down), cache vẫn tự hết hạn sau khoảng ngắn thay vì kẹt cả giờ như base
    Note over Metric: Request đọc kế tiếp sẽ miss cache và rebuild atomic theo version mới (xem flow "View product page")
```
