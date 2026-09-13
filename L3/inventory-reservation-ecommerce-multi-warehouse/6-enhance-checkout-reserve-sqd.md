# Sequence - Enhance - Flow "checkout-reserve"

Đây là **enhance**, cùng flow `checkout-reserve` như ở base nhưng thay đọc-rồi-ghi bằng update nguyên tử `UPDATE inventory SET reserved = reserved + 1 WHERE available - reserved > 0`, đáp ứng yêu cầu 1: chỉ 1 trong 2 khách giữ được hàng, người thua nhận thông báo ngay và được đề xuất kho khác hoặc báo hết hàng. Mỗi lần giữ thành công đều tạo 1 dòng `Reservation` có TTL riêng, đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    actor CustomerA as Khách A (khu vực gần kho X)
    actor CustomerB as Khách B (khu vực gần kho X)
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: available = 1, reserved = 0 tại kho X

    par Khách A vào checkout
        CustomerA->>App: vào checkout sản phẩm Y
        App->>Inventory: UPDATE reserved = reserved + 1 WHERE available - reserved > 0
        Inventory-->>App: 1 dòng bị ảnh hưởng, thành công
        App->>App: tạo Reservation, status = active, expires_at = now + 15 phút
        App-->>CustomerA: giữ hàng thành công tại kho X
    and Khách B vào checkout gần như cùng lúc
        CustomerB->>App: vào checkout sản phẩm Y
        App->>Inventory: UPDATE reserved = reserved + 1 WHERE available - reserved > 0
        Inventory-->>App: 0 dòng bị ảnh hưởng, điều kiện không còn đúng
        App-->>App: tìm kho khác còn hàng gần khách B
        alt Có kho khác còn hàng
            App-->>CustomerB: đề xuất giữ hàng ở kho khác (thời gian giao có thể lâu hơn)
        else Không còn kho nào khác
            App-->>CustomerB: thông báo hết hàng ngay lập tức
        end
    end
```
