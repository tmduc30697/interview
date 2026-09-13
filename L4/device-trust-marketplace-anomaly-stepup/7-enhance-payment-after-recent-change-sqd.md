# Enhance sequence — Tự động yêu cầu xác minh lại khi thanh toán ngay sau khi vừa đổi thông tin nhạy cảm

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Ngay cả khi session không bị gắn cờ nghi ngờ và người dùng đã xác minh xong cho lần đổi địa chỉ/phương thức thanh toán trước đó, hệ thống vẫn tự động buộc xác minh lại thêm 1 lần nữa nếu thanh toán diễn ra ngay sau đó, để chặn kịch bản chiếm session rồi đổi địa chỉ nhận hàng rồi thanh toán liền. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ chiếm session (hoặc User hợp lệ)
    participant App as Marketplace App
    participant Verify as Verification Service
    participant DB as Database

    Attacker->>App: Đổi địa chỉ giao hàng sang địa chỉ lạ
    App->>Verify: Yêu cầu xác minh cho change_address, xác minh thành công
    Verify->>DB: UPDATE ADDRESS (updated_at=now())

    Note over Attacker,App: Chỉ 30 giây sau, cùng session này thực hiện thanh toán

    Attacker->>App: Nhấn thanh toán đơn hàng giá trị lớn
    App->>DB: Kiểm tra ADDRESS.updated_at và PAYMENT_METHOD.updated_at của đơn hàng này
    DB-->>App: address vừa được đổi cách đây 30 giây, nằm trong ngưỡng "vừa thay đổi gần đây" (vd dưới 15 phút)

    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=checkout_payment, trigger_reason=payment_after_recent_change)
    Note over App,Verify: Bắt buộc xác minh lại từ đầu, KHÔNG tính là đã xác minh trong cùng phiên dù lần đổi địa chỉ trước đó vừa xác minh xong

    Verify->>Attacker: Yêu cầu xác minh bổ sung trước khi hoàn tất thanh toán
    alt Xác minh thất bại (không đúng OTP, không phải sinh trắc học của chủ tài khoản)
        Verify-->>App: Từ chối, không cho hoàn tất thanh toán
        App-->>Attacker: Giao dịch bị chặn, tài khoản thật không mất tiền
    else Xác minh thành công (đúng là chủ tài khoản)
        Verify->>DB: INSERT ORDER (status=paid)
        DB-->>App: Thanh toán thành công
        App-->>Attacker: Đơn hàng hoàn tất
    end
```
