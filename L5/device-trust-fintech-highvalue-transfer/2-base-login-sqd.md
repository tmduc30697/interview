# Base sequence — Login (không phân biệt mức trust thiết bị)

Đây là **base**, flow đăng nhập ở trạng thái hiện tại: sau khi xác thực đúng mật khẩu/MFA, thiết bị (mới hay cũ) đều được cấp toàn quyền truy cập như nhau, hệ thống chỉ ghi nhận fingerprint thiết bị mà không gán mức trust nào. Đây là nền để so sánh với yêu cầu 1 và 3 của đề bài (cần phân tầng trust và phân biệt thiết bị mới thật sự với thiết bị cài lại app).

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant Auth as Auth Service
    participant DB as Database

    User->>App: Nhập username/password
    App->>Auth: Xác thực password + MFA (OTP/app authenticator)
    Auth-->>App: Xác thực thành công

    App->>DB: INSERT/UPDATE DEVICE (device_fingerprint, first_seen_at nếu chưa có)
    DB-->>App: Ghi nhận thiết bị, không gán mức trust nào

    Auth-->>App: Trả access token, toàn quyền như mọi thiết bị khác
    App-->>User: Đăng nhập thành công, có thể xem số dư và chuyển tiền ngay không giới hạn
```
