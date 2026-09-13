# Base sequence — Checkout (đổi địa chỉ, đổi phương thức thanh toán, thanh toán đều chỉ cần session còn hạn)

Đây là **base**, flow thanh toán ở trạng thái hiện tại: chỉ cần session "remember me" còn hạn là đủ để đổi địa chỉ giao hàng, đổi phương thức thanh toán, và hoàn tất thanh toán, không có bước xác thực bổ sung nào dù đây đều là hành động nhạy cảm. Sơ đồ minh hoạ đúng khoảng trống mà yêu cầu 1 và 4 của đề bài nhắm tới: nếu 1 session bị chiếm quyền, kẻ tấn công có thể đổi địa chỉ nhận hàng rồi thanh toán ngay mà không gặp trở ngại nào.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant DB as Database

    User->>App: Mở app với session remember-me còn hạn
    App->>DB: Kiểm tra SESSION còn hạn (expires_at > now)
    DB-->>App: Session hợp lệ

    User->>App: Đổi địa chỉ giao hàng sang địa chỉ mới
    App->>DB: UPDATE ADDRESS, chỉ cần session còn hạn, không hỏi lại gì thêm
    DB-->>App: Đã cập nhật địa chỉ

    User->>App: Đổi phương thức thanh toán sang thẻ mới
    App->>DB: UPDATE PAYMENT_METHOD, chỉ cần session còn hạn
    DB-->>App: Đã cập nhật phương thức thanh toán

    User->>App: Nhấn thanh toán đơn hàng
    App->>DB: INSERT ORDER (status=paid), chỉ cần session còn hạn
    DB-->>App: Thanh toán thành công ngay lập tức
    App-->>User: Đơn hàng hoàn tất, giao tới địa chỉ vừa đổi, trả bằng thẻ vừa đổi
```
