# Base sequence — Forgot password (chưa hardening)

Đây là **base**, flow "Quên mật khẩu" ở trạng thái thô sơ hiện tại — chính flow này là đối tượng mà toàn bộ 5 yêu cầu của đề bài nhắm vào để siết lại.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant DB as USER / PASSWORD_RESET_TOKEN store
    participant Email as Email Gateway

    User->>App: Nhập email, bấm "Quên mật khẩu"
    App->>DB: Tìm USER theo email
    alt Email tồn tại
        DB-->>App: Tìm thấy user
        App->>DB: Tạo PASSWORD_RESET_TOKEN mới (không hạn dùng, không đánh dấu one-time)
        App->>Email: Gửi link chứa token
        App-->>User: "Link reset đã được gửi"
    else Email không tồn tại
        DB-->>App: Không tìm thấy
        App-->>User: "Email không tồn tại trong hệ thống"
        Note over App: Lộ thông tin email có tồn tại hay không (account enumeration)
    end
    User->>App: Mở link, nhập mật khẩu mới
    App->>DB: Kiểm tra token có tồn tại (không kiểm tra hạn dùng, không kiểm tra đã dùng chưa)
    DB-->>App: Token hợp lệ
    App->>DB: Cập nhật password_hash
    App-->>User: Đổi mật khẩu thành công
    Note over DB: Token cũ vẫn còn dùng được lại, các SESSION khác không bị đăng xuất, không có email thông báo, không rate-limit
```
