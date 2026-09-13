# Enhance sequence — Checkout (revalidate giá/tồn kho trước khi thanh toán)

Đây là **enhance**, cùng flow "checkout" như ở base nhưng thêm bước revalidate: vì giá dùng ở giỏ hàng có thể là `price_snapshot`/`price_at_add` đã cũ (đặc biệt với guest cart lưu lâu trong localStorage), trước khi xác nhận thanh toán hệ thống phải kiểm tra lại giá/tồn kho thật với server và xử lý rõ khi giá đổi hoặc hết hàng, không để user thanh toán nhầm giá cũ. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Checkout UI
    participant API as Order API
    participant DB as Server DB
    participant Reval as PRICE_REVALIDATION

    User->>SPA: Mở trang checkout
    SPA->>SPA: Lấy danh sách item và price_snapshot/price_at_add hiện có
    User->>SPA: Điền/xác nhận địa chỉ, ghi chú
    User->>SPA: Bấm đặt hàng

    SPA->>API: Revalidate giá và tồn kho thật cho từng item trước khi submit order
    API->>DB: Lấy giá và stock hiện tại của từng sản phẩm
    DB-->>API: Trả về current_price, current_stock
    API-->>SPA: Trả kết quả so sánh với price_snapshot

    loop Với mỗi item
        alt Giá và tồn kho không đổi
            SPA->>Reval: Ghi PRICE_REVALIDATION status=unchanged
        else Giá đã thay đổi
            SPA->>Reval: Ghi PRICE_REVALIDATION status=price_changed
            SPA-->>User: Cảnh báo rõ giá đã thay đổi từ price_snapshot sang current_price, yêu cầu xác nhận lại
        else Hết hàng
            SPA->>Reval: Ghi PRICE_REVALIDATION status=out_of_stock
            SPA-->>User: Cảnh báo sản phẩm đã hết hàng, yêu cầu xoá khỏi giỏ hoặc chọn số lượng khác
        end
    end

    alt Toàn bộ item hợp lệ sau revalidate hoặc user đã xác nhận giá mới
        SPA->>API: Submit order với current_price đã revalidate
        API->>DB: Tạo đơn hàng
        API-->>SPA: Xác nhận đặt hàng thành công
    else Còn item chưa xử lý (hết hàng chưa gỡ, giá mới chưa xác nhận)
        SPA-->>User: Chặn đặt hàng cho tới khi xử lý xong các cảnh báo
    end
```
