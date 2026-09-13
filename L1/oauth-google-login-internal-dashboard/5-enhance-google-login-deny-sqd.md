# Enhance sequence — Google login deny

Đây là **enhance**, flow hoàn toàn mới so với base — xử lý trường hợp user bấm "Deny" ở màn hình consent của Google, đúng yêu cầu cuối trong đề bài. Chọn vẽ riêng flow này vì đây là 1 nhánh kết thúc sớm, không đi tới bước đổi code lấy token, cần xử lý gọn để không văng lỗi khó hiểu cho user.

```mermaid
sequenceDiagram
    actor User
    participant WebApp
    participant AuthServer
    participant DB as Database
    participant Google as Google OAuth

    User->>WebApp: Bấm "Sign in with Google"
    WebApp->>AuthServer: GET /auth/google/start
    AuthServer->>DB: Sinh và lưu OAUTH_STATE (state, redirect_uri)
    AuthServer-->>WebApp: authorization_url kèm state
    WebApp->>Google: Redirect user tới authorization_url
    Google-->>User: Màn hình đăng nhập + consent
    User->>Google: Bấm "Deny" ở màn hình consent
    Google-->>WebApp: Redirect về callback kèm error=access_denied, state
    WebApp->>AuthServer: GET /auth/google/callback { error, state }
    AuthServer->>DB: Đối chiếu state, đánh dấu OAUTH_STATE đã dùng
    Note over AuthServer: Không có code nào để đổi lấy token, dừng flow ngay tại đây
    AuthServer-->>WebApp: Thông báo user đã từ chối cấp quyền
    WebApp-->>User: Hiển thị lại màn hình login, cho phép thử lại hoặc dùng email/password
```
