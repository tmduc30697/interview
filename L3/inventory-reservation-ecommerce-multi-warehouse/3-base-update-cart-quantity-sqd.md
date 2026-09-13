# Sequence - Base - Flow "update-cart-quantity"

Đây là **base**: khi khách sửa số lượng trong giỏ hàng lúc đang giữ reservation, hệ thống hủy phần đã giữ rồi giữ lại từ đầu theo số lượng mới - 2 bước tách rời. Flow này được chọn vì đây chính là cách làm mà yêu cầu 4 chỉ ra là có vấn đề: giữa lúc hủy và giữ lại, tồn kho đã giữ có thể bị người khác giành mất.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: reserved = 1 (do khách này giữ số lượng 1 trước đó)

    Customer->>App: sửa số lượng trong giỏ từ 1 lên 3
    App->>Inventory: bước 1, giải phóng reserved -= 1 (reserved về 0)

    Note over Inventory: Khoảng hở giữa 2 bước, người khác có thể giành tồn kho ngay lúc này

    App->>Inventory: bước 2, giữ lại reserved += 3 cho số lượng mới

    alt Không ai giành tồn kho trong khoảng hở
        Inventory-->>App: giữ thành công 3 đơn vị
        App-->>Customer: cập nhật giỏ hàng thành công
    else Khách khác giành mất tồn kho trong khoảng hở
        Inventory-->>App: không đủ tồn kho để giữ 3 đơn vị
        App-->>Customer: báo lỗi, tồn kho đã bị giành dù khách này đã giữ trước đó
    end
```
