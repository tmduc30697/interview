# Enhance sequence — Login (bắt buộc enroll MFA trước khi dùng bất kỳ tính năng nào)

Đây là **enhance** của flow `login` đã có ở base. So với base (chỉ password, không MFA), nay user mới không được phép "bỏ qua để sau" — phải hoàn tất enroll ít nhất 1 phương thức MFA (TOTP hoặc SMS backup) ngay sau khi đăng nhập lần đầu, trước khi truy cập bất kỳ tính năng nào của app. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (lần đầu đăng nhập)
    participant Server
    participant DB as Database

    U->>Server: POST /login (phone_number, password)
    Server->>DB: Xác thực password
    DB-->>Server: OK
    Server->>DB: SELECT USER.onboarding_completed
    DB-->>Server: false, chưa enroll MFA nào

    Server-->>U: Bắt buộc enroll MFA trước, chặn mọi tính năng khác, không có nút "bỏ qua"

    U->>Server: Chọn enroll TOTP app, quét QR code
    Server->>DB: INSERT MFA_FACTOR (type=totp, is_primary=true)
    DB-->>Server: OK
    Server-->>U: Yêu cầu nhập mã TOTP để xác nhận enroll đúng

    U->>Server: Nhập mã TOTP đúng
    Server-->>U: Gợi ý thêm SMS backup (khuyến nghị, không bắt buộc thêm phương thức thứ 2)
    U->>Server: Thêm số điện thoại nhận SMS backup
    Server->>DB: INSERT MFA_FACTOR (type=sms, is_primary=false)
    Server->>DB: UPDATE USER SET onboarding_completed=true
    DB-->>Server: OK

    Server-->>U: Hoàn tất, được phép truy cập toàn bộ tính năng app

    Note over U,Server: Ở lần đăng nhập sau, user chỉ cần password + xác thực bằng factor đã enroll (TOTP chính, hoặc SMS backup)
```
