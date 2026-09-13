# Base sequence — Login

Đây là **base**, flow "Đăng nhập bằng email/password" — tiền đề bắt buộc cho enhance, vì "Sign in with Google" được thêm để thay thế/song song với chính flow này. Chọn flow này để làm rõ cơ chế phát session token nội bộ đã có sẵn trước khi có thêm phương thức đăng nhập qua Google.

```mermaid
sequenceDiagram
    actor User
    participant WebApp
    participant AuthServer
    participant DB as Database

    User->>WebApp: Nhập email và password
    WebApp->>AuthServer: POST /login { email, password }
    AuthServer->>DB: Tìm user theo email, kiểm tra password_hash
    DB-->>AuthServer: user hợp lệ
    AuthServer->>DB: Tạo session
    DB-->>AuthServer: session_token
    AuthServer-->>WebApp: session_token (JWT/cookie)
    WebApp-->>User: Vào dashboard
```
