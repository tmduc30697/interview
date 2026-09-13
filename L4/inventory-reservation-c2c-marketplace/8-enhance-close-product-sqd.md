# Sequence - Enhance - Flow "close-product"

Đây là **enhance**, cùng flow `close-product` như ở base nhưng thay đổi hành vi cốt lõi, đáp ứng yêu cầu 2: ranh giới "trước/sau" được định nghĩa rõ bằng `reservation.created_at` so với `product.closed_at`, không phụ thuộc vào thời điểm request của buyer chạm tới server. Buyer đã giữ reservation hợp lệ trước mốc đóng vẫn được mua, buyer mới sau mốc đó không thể tạo reservation.

```mermaid
sequenceDiagram
    actor BuyerOld as Buyer đã giữ reservation trước đó
    actor BuyerNew as Buyer mới
    actor Seller
    participant App as Marketplace App
    participant DB as Database
    participant AuditLog as Inventory Audit Log

    Note over BuyerOld: reservation.created_at = 10:00:00, status = active

    Seller->>App: Ngừng bán sản phẩm X lúc 10:00:05
    App->>DB: UPDATE product SET status = closed, closed_at = 10:00:05
    App->>AuditLog: ghi log, changed_by = Seller, field = status, old_value = active, new_value = closed

    BuyerOld->>App: xác nhận đặt mua lúc 10:00:10 (dùng reservation tạo lúc 10:00:00)
    App->>DB: kiểm tra reservation.created_at (10:00:00) so với product.closed_at (10:00:05)
    Note over App: reservation.created_at < closed_at => hợp lệ theo mốc thời điểm tạo
    App->>DB: reservation.status = confirmed
    App-->>BuyerOld: xác nhận đặt mua thành công

    BuyerNew->>App: cố tạo reservation mới cho sản phẩm X lúc 10:00:20
    App->>DB: kiểm tra product.status
    Note over App: product.status = closed => không cho tạo reservation mới
    App-->>BuyerNew: từ chối, sản phẩm đã ngừng bán
```
