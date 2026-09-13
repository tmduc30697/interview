# Enhance sequence — Update MFA device (tự yêu cầu MFA hiện tại + cảnh báo email ngay)

Đây là **enhance**, flow mới phát sinh từ đề bài — mọi thay đổi liên quan tới MFA (thêm/xóa thiết bị, đổi số điện thoại nhận OTP) phải tự nó yêu cầu xác thực MFA hiện tại trước khi thực hiện, và gửi cảnh báo qua email ngay khi thay đổi hoàn tất. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor U as User
    participant Server
    participant DB as Database
    participant EmailService as Dịch vụ gửi email

    U->>Server: Yêu cầu đổi số điện thoại nhận SMS OTP
    Server-->>U: Yêu cầu xác thực MFA hiện tại trước khi cho đổi

    U->>Server: Nhập mã OTP từ TOTP app (factor hiện tại, không phải factor sắp bị đổi)
    Server->>DB: Xác thực OTP hợp lệ

    alt Xác thực MFA thành công
        Server->>DB: UPDATE MFA_FACTOR SET phone_number=<số mới> WHERE type=sms, user=U
        Server->>DB: INSERT MFA_CHANGE_AUDIT_LOG (action=change_phone_number, performed_at=now())
        DB-->>Server: OK

        Server->>EmailService: Gửi email cảnh báo ngay tới địa chỉ email đã đăng ký của U
        EmailService-->>U: "Số điện thoại nhận OTP của bạn vừa được đổi lúc <thời điểm>, nếu không phải bạn hãy liên hệ ngay"

        Server->>DB: UPDATE MFA_CHANGE_AUDIT_LOG SET email_alert_sent=true
        Server-->>U: Đổi số điện thoại thành công
    else Xác thực MFA thất bại
        Server-->>U: Từ chối thay đổi, số điện thoại cũ vẫn giữ nguyên
    end
```
