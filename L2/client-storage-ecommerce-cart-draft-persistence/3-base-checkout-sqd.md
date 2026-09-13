# Base sequence — Checkout (form nhập trực tiếp, không lưu nháp, giá lấy live)

Đây là **base**, flow checkout ở trạng thái trước enhance. Người dùng điền địa chỉ/ghi chú trực tiếp trên form, không có bước lưu nháp nào — nếu thoát giữa chừng thì mất hết thông tin đã nhập. Giá sản phẩm được lấy trực tiếp từ CART_ITEM đã ghi ở bước thêm giỏ hàng, dùng luôn để đặt hàng, không có bước revalidate lại. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi thêm revalidate giá/tồn kho và cơ chế TTL cho form nháp.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Checkout UI
    participant API as Order API
    participant DB as Server DB

    User->>SPA: Mở trang checkout
    SPA->>API: Lấy CART_ITEM hiện tại (product, quantity, price_at_add)
    API-->>SPA: Trả về danh sách item và giá đã ghi từ lúc thêm giỏ
    User->>SPA: Điền địa chỉ, ghi chú trực tiếp trên form
    Note over SPA: Không có bước lưu nháp, nếu thoát giữa chừng sẽ mất thông tin đã nhập
    User->>SPA: Bấm đặt hàng
    SPA->>API: Submit order với giá price_at_add đã có, không kiểm tra lại giá/tồn kho hiện tại
    API->>DB: Tạo đơn hàng
    API-->>SPA: Xác nhận đặt hàng thành công
```
