# Enhance sequence — Checkout draft tự hết hạn theo TTL

Đây là **enhance**, flow hoàn toàn mới, xử lý form checkout dở dang (địa chỉ, ghi chú) lưu trong localStorage. Vì thông tin này có thể lỗi thời (địa chỉ cũ, khuyến mãi hết hạn) nếu để lâu, mỗi bản nháp phải có TTL (ví dụ 24h) và tự động không được coi là dữ liệu hiện tại khi đã hết hạn. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant SPA as SPA Checkout UI
    participant LS as localStorage (CHECKOUT_DRAFT_LOCAL)

    User->>SPA: Điền địa chỉ, ghi chú trên form checkout
    SPA->>LS: Lưu CHECKOUT_DRAFT_LOCAL(address, note, saved_at, expires_at=saved_at+24h)
    Note over User,SPA: User thoát trang giữa chừng, chưa đặt hàng

    User->>SPA: Quay lại trang checkout sau đó
    SPA->>LS: Đọc CHECKOUT_DRAFT_LOCAL nếu có
    alt Còn hạn (now < expires_at)
        LS-->>SPA: Trả về address, note đã lưu
        SPA-->>User: Điền sẵn lại form với dữ liệu nháp còn hiệu lực
    else Đã hết hạn (now >= expires_at, ví dụ quá 24h)
        LS-->>SPA: Draft coi như không hợp lệ nữa
        SPA->>LS: Xoá CHECKOUT_DRAFT_LOCAL đã hết hạn
        SPA-->>User: Hiển thị form trống, không dùng địa chỉ/khuyến mãi cũ như dữ liệu hiện tại
    end
```
