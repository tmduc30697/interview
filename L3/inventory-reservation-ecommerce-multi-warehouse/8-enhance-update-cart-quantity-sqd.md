# Sequence - Enhance - Flow "update-cart-quantity"

Đây là **enhance**, cùng flow `update-cart-quantity` như ở base nhưng thay hủy-rồi-giữ-lại bằng 1 update nguyên tử tính đúng phần chênh lệch, đáp ứng yêu cầu 4: khi khách sửa số lượng từ 1 lên 3, hệ thống chỉ cần giữ thêm phần chênh lệch (2 đơn vị) trong cùng 1 thao tác, không có khoảng hở nào để người khác giành mất tồn kho đã giữ.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: Reservation hiện tại của khách này, quantity = 1, status = active

    Customer->>App: sửa số lượng trong giỏ từ 1 lên 3
    App-->>App: tính delta = 3 - 1 = 2 (phần cần giữ thêm)

    App->>Inventory: UPDATE reserved = reserved + 2 WHERE available - reserved >= 2
    alt Đủ tồn kho cho phần chênh lệch
        Inventory-->>App: thành công
        App->>App: cập nhật Reservation.quantity = 3 trong cùng transaction
        App-->>Customer: cập nhật giỏ hàng thành công, vẫn giữ nguyên phần đã có
    else Không đủ tồn kho cho phần chênh lệch
        Inventory-->>App: thất bại, không đủ 2 đơn vị còn lại
        App-->>Customer: báo không thể tăng số lượng, giữ nguyên reservation cũ (vẫn còn 1)
    end
```
