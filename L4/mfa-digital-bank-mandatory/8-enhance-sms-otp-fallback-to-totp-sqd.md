# Enhance sequence — SMS OTP fallback to TOTP (không để user bị kẹt ngoài tài khoản)

Đây là **enhance**, flow mới phát sinh từ đề bài — khi gửi OTP qua SMS thất bại hoặc chậm, hệ thống cung cấp phương án fallback rõ ràng sang TOTP app thay vì để user bị kẹt hoàn toàn ngoài tài khoản. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (chỉ chọn SMS làm phương thức chính lúc login)
    participant Server
    participant SMSProvider as Nhà cung cấp SMS
    participant DB as Database

    Server->>DB: INSERT OTP_CODE (channel=sms, purpose=login, expires_at=now()+5 phút)
    Server->>SMSProvider: Gửi mã OTP tới số điện thoại của U
    SMSProvider-->>Server: Lỗi gửi thất bại, hoặc không phản hồi sau 30 giây

    Server->>DB: UPDATE OTP_CODE SET status=expired (do kênh gửi thất bại)
    Server-->>U: "Không gửi được SMS, bạn có thể dùng TOTP app (nếu đã enroll) để xác thực thay thế"

    alt User đã enroll TOTP app trước đó (is_primary hoặc backup)
        Server->>DB: INSERT OTP_CODE mới (channel=totp, purpose=login)
        Server-->>U: Nhập mã từ TOTP app
        U->>Server: Nhập mã TOTP đúng
        Server->>DB: UPDATE OTP_CODE SET status=verified
        Server-->>U: Đăng nhập thành công qua kênh fallback
    else User chưa enroll TOTP, chỉ có SMS
        Server-->>U: Thử gửi lại SMS qua nhà cung cấp dự phòng thứ 2, hoặc hướng dẫn liên hệ hotline xác minh danh tính thủ công
        Note over Server: Không để user hoàn toàn không có cách nào truy cập tài khoản
    end
```
