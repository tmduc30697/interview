# Base sequence — Add to cart (guest cart chỉ tồn tại trong bộ nhớ, mất khi reload)

Đây là **base**, flow thêm sản phẩm vào giỏ ở trạng thái trước enhance. Với user đã đăng nhập, item được ghi thẳng lên giỏ hàng server. Với guest (chưa đăng nhập), giỏ hàng chỉ tồn tại trong biến JS ở bộ nhớ trang, không có bước lưu nào ra localStorage — nên reload hoặc đóng nhầm tab là mất toàn bộ giỏ. Đây chính là vấn đề mà enhance sẽ giải quyết bằng localStorage.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant Mem as In-memory cart (JS variable, guest)
    participant API as Cart API
    participant DB as Server DB

    User->>SPA: Bấm thêm sản phẩm vào giỏ
    alt Đã đăng nhập
        SPA->>API: POST thêm CART_ITEM
        API->>DB: Lưu CART_ITEM(product_id, quantity, price_at_add)
        API-->>SPA: Xác nhận, trả về giỏ hàng cập nhật
    else Guest (chưa đăng nhập)
        SPA->>Mem: Thêm item vào biến giỏ hàng tạm trong bộ nhớ
        Note over Mem: Không có bước lưu nào ra localStorage, chỉ tồn tại trong phiên trang hiện tại
        Mem-->>SPA: Cập nhật badge số lượng trên UI
    end
    Note over SPA,Mem: Nếu guest reload trang hoặc đóng nhầm tab, toàn bộ giỏ hàng trong bộ nhớ bị mất
```
