# Sequence - Enhance - Flow "instore-sale-sync"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 5: khi kênh bán tại cửa hàng vật lý dùng chung tồn kho biến thể với kênh online, một biến thể vừa bán hết tại cửa hàng phải được đồng bộ kịp thời về tồn kho online của đúng biến thể đó, tránh nhận đơn online cho hàng thực tế đã không còn.

```mermaid
sequenceDiagram
    participant POS as POS cửa hàng vật lý
    participant SyncService as Omni-channel Sync Service
    participant Variant as Variant (size M, màu đen)
    actor OnlineCustomer as Khách mua online

    POS->>SyncService: bán hết size M, màu đen tại cửa hàng, sold_quantity = 1
    SyncService->>Variant: UPDATE quantity = quantity - 1 WHERE product_id, size, color khớp
    SyncService-->>SyncService: ghi InstoreSaleEvent, sold_at = now

    Note over SyncService,Variant: Độ trễ đồng bộ được quy định ở mức gần thời gian thực,\nví dụ dưới vài giây kể từ khi POS xác nhận bán

    SyncService->>Variant: cập nhật last_synced_at sau khi đồng bộ xong
    SyncService-->>SyncService: ghi synced_to_online_at vào InstoreSaleEvent để đo độ trễ thực tế

    OnlineCustomer->>SyncService: xem trang sản phẩm, chọn size M màu đen, bấm mua
    SyncService->>Variant: truy vấn tồn kho khả dụng thực (đã được đồng bộ từ POS)
    Variant-->>SyncService: quantity = 0

    SyncService-->>OnlineCustomer: báo hết hàng đúng biến thể,\ntránh nhận đơn online cho hàng đã bán tại cửa hàng vật lý

    Note over SyncService: Nếu độ trễ đồng bộ vượt ngưỡng cho phép,\nhệ thống cảnh báo vận hành vì rủi ro oversell tăng cao
```
