# Enhance sequence — Checkout (hành động nhạy cảm luôn cần xác minh bổ sung dù session còn hạn)

Đây là **enhance** của flow `checkout` đã có ở base. So với base (chỉ cần session còn hạn là đổi địa chỉ/phương thức thanh toán/thanh toán được ngay), nay mỗi hành động nhạy cảm đều tạo 1 `SENSITIVE_ACTION_VERIFICATION` riêng và bắt buộc xác minh lại (vd mã OTP, xác nhận sinh trắc học) trước khi được thực hiện, bất kể session chính vẫn còn hạn. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant Verify as Verification Service
    participant DB as Database

    User->>App: Mở app với session remember-me còn hạn
    App->>DB: Kiểm tra SESSION còn hạn và status=active
    DB-->>App: Session hợp lệ, cho phép duyệt sản phẩm bình thường

    User->>App: Đổi địa chỉ giao hàng sang địa chỉ mới
    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=change_address, trigger_reason=sensitive_action)
    Verify->>User: Yêu cầu xác minh lại (OTP gửi qua kênh đã đăng ký)
    User-->>Verify: Nhập đúng OTP
    Verify->>DB: UPDATE SENSITIVE_ACTION_VERIFICATION status=verified
    Verify->>DB: UPDATE ADDRESS (updated_at=now())
    DB-->>App: Địa chỉ đã được cập nhật

    User->>App: Đổi phương thức thanh toán sang thẻ mới
    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=change_payment_method, trigger_reason=sensitive_action)
    Verify->>User: Yêu cầu xác minh lại lần nữa
    User-->>Verify: Nhập đúng OTP
    Verify->>DB: UPDATE SENSITIVE_ACTION_VERIFICATION status=verified
    Verify->>DB: UPDATE PAYMENT_METHOD (updated_at=now())
    DB-->>App: Phương thức thanh toán đã được cập nhật

    User->>App: Nhấn thanh toán đơn hàng
    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=checkout_payment, trigger_reason=sensitive_action)
    Verify->>User: Yêu cầu xác minh lại trước khi hoàn tất thanh toán
    User-->>Verify: Xác minh thành công
    Verify->>DB: INSERT ORDER (status=paid)
    DB-->>App: Thanh toán thành công
    App-->>User: Đơn hàng hoàn tất

    Note over App,Verify: Dù session dài hạn còn hạn tới 30 ngày, mỗi hành động nhạy cảm vẫn đòi hỏi 1 lần xác minh riêng, tách biệt hoàn toàn khỏi trạng thái "đã đăng nhập"
```
