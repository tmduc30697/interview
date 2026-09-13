# Sequence - Enhance - Flow "stock-display-sync"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 5: đồng bộ giữa tồn kho hiển thị công khai trên trang sản phẩm và tồn kho khả dụng thực (`available - reserved`), quy định độ trễ chấp nhận được và tránh hiển thị "còn hàng" khi thực chất đã được reserve hết bởi các giỏ hàng khác.

```mermaid
sequenceDiagram
    actor Shopper as Khách đang xem trang sản phẩm
    participant App as E-commerce App
    participant Inventory as Inventory (tổng nhiều kho)
    participant Display as Product Stock Display
    participant SyncJob as Stock Display Sync Job

    loop Mỗi vài giây (độ trễ chấp nhận được, ví dụ tối đa 5 giây)
        SyncJob->>Inventory: tính available - reserved theo từng sản phẩm, gộp tất cả kho
        Inventory-->>SyncJob: available_thuc_te
        SyncJob->>Display: cập nhật displayed_available = available_thuc_te,\nsynced_at = now, sync_lag_seconds = độ trễ đo được
    end

    Shopper->>App: xem trang sản phẩm Y
    App->>Display: đọc displayed_available (snapshot đã đồng bộ gần nhất)
    Display-->>App: displayed_available = 0 (đã bị reserve hết bởi giỏ hàng khác)
    App-->>Shopper: hiển thị "Hết hàng" thay vì "Còn hàng" dựa trên số cũ đã lỗi thời

    Note over Display: Nếu sync_lag_seconds vượt ngưỡng chấp nhận được,\nhệ thống cảnh báo để tránh hiển thị sai kéo dài
```
