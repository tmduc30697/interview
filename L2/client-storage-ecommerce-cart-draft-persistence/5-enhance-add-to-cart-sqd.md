# Enhance sequence — Add to cart (guest cart bền vững qua localStorage)

Đây là **enhance**, cùng flow "add-to-cart" như ở base nhưng thay đổi ở phần guest: thay vì chỉ giữ trong biến bộ nhớ JS (mất khi reload), item được ghi vào localStorage kèm `price_snapshot` tại thời điểm thêm. Đây là nền cho yêu cầu 1 (merge khi đăng nhập) và yêu cầu 2 (snapshot giá để revalidate khi checkout); việc phát sự kiện đồng bộ đa tab được nói rõ hơn ở flow multi-tab-cart-sync riêng.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Cart UI
    participant LS as localStorage (LOCAL_CART_ITEM)
    participant API as Cart API
    participant DB as Server DB

    User->>SPA: Bấm thêm sản phẩm vào giỏ
    alt Đã đăng nhập
        SPA->>API: POST thêm CART_ITEM
        API->>DB: Lưu CART_ITEM(product_id, quantity, price_at_add)
        API-->>SPA: Xác nhận, trả về giỏ hàng cập nhật
    else Guest (chưa đăng nhập)
        SPA->>LS: Ghi/cập nhật LOCAL_CART_ITEM(product_id, quantity, price_snapshot=giá hiện tại, updated_at)
        LS-->>SPA: Ghi thành công
        Note over LS: Giỏ hàng vẫn còn sau khi reload hoặc mở lại tab, khác với base
        SPA->>SPA: Phát sự kiện đồng bộ cho các tab khác (chi tiết ở flow multi-tab-cart-sync)
    end
```
