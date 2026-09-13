# Enhance sequence — Google login

Đây là **enhance**, flow hoàn toàn mới so với base — đăng nhập qua "Sign in with Google" bằng authorization code flow. Đây là flow trung tâm của enhance, thể hiện state chống CSRF + validate redirect_uri, giới hạn theo domain công ty, account linking khi email đã tồn tại, và việc access token của Google chỉ dùng 1 lần rồi không lưu lại.

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
    AuthServer-->>WebApp: authorization_url kèm state, redirect_uri
    WebApp->>Google: Redirect user tới authorization_url
    Google-->>User: Màn hình đăng nhập + consent
    User->>Google: Đăng nhập Google và đồng ý (Allow)
    Google-->>WebApp: Redirect về callback kèm code, state
    WebApp->>AuthServer: GET /auth/google/callback { code, state }
    AuthServer->>DB: Đối chiếu state với OAUTH_STATE đã lưu, kiểm tra redirect_uri khớp whitelist
    Note over AuthServer,DB: state không khớp hoặc đã dùng rồi thì từ chối ngay, chặn CSRF
    AuthServer->>Google: POST /token { code } đổi lấy access_token
    Google-->>AuthServer: access_token (dùng 1 lần)
    AuthServer->>Google: GET /userinfo (access_token)
    Google-->>AuthServer: profile { email, sub, hosted_domain }
    Note over AuthServer: access_token bị huỷ ngay sau bước này, không lưu lại
    alt domain không thuộc công ty
        AuthServer-->>WebApp: 403 domain không được phép đăng nhập
        WebApp-->>User: Hiển thị lỗi domain không hợp lệ
    else domain hợp lệ
        AuthServer->>DB: Tìm GOOGLE_IDENTITY theo google_sub
        alt đã có GOOGLE_IDENTITY
            DB-->>AuthServer: user đã liên kết
        else chưa có, kiểm tra email đã tồn tại
            AuthServer->>DB: Tìm USER theo email
            alt email đã tồn tại (tạo bằng password trước đó)
                AuthServer->>DB: Tạo GOOGLE_IDENTITY, map vào user cũ (account linking)
            else email chưa tồn tại
                AuthServer->>DB: Tạo USER mới + GOOGLE_IDENTITY tương ứng
            end
        end
        AuthServer->>DB: Tạo SESSION cho user
        DB-->>AuthServer: session_token nội bộ
        AuthServer-->>WebApp: session_token (JWT/cookie)
        WebApp-->>User: Vào dashboard
    end
```
