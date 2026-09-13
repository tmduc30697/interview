# "Đăng nhập bằng Google" cho một SaaS dashboard nội bộ

**Hệ thống:** Một dashboard quản lý task nội bộ (kiểu Trello mini) cho công ty, hiện chỉ có đăng nhập bằng email/password.

**Vai trò của flow:** Thêm nút "Sign in with Google" — app đóng vai trò OAuth Client, dùng authorization code flow để lấy identity token và định danh user, thay thế/song song với login truyền thống.

**Yêu cầu cụ thể:**
- Chỉ cho phép đăng nhập bằng các Google account thuộc domain công ty (chặn domain khác).
- Lần đầu login qua Google phải tự tạo user record và map với account cũ nếu email đã tồn tại (account linking).
- Phải dùng `state` param để chống CSRF, và validate `redirect_uri` khớp whitelist.
- Access token của Google chỉ dùng một lần để lấy profile — không lưu lại; hệ thống tự phát session token riêng (JWT/session cookie) sau khi xác thực xong.
- Xử lý được case user bấm "Deny" ở màn hình consent của Google.

---

# Base ERD — Dashboard nội bộ chỉ có login email/password

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài. Đề bài nói rõ hệ thống "hiện chỉ có đăng nhập bằng email/password" — nên base chỉ cần entity xoay quanh User + session đăng nhập truyền thống, không bịa thêm các entity khác của dashboard quản lý task (board, card...) vì đề bài không liên quan tới chúng. Base **chưa có** khái niệm OAuth identity liên kết Google, chưa có state chống CSRF, chưa có logic giới hạn theo domain công ty — tất cả là phần enhance.

```mermaid
erDiagram
    USER ||--o{ SESSION : "đăng nhập"

    USER {
        string id PK
        string email
        string password_hash
        string name
        datetime created_at
    }
    SESSION {
        string id PK
        string user_id FK
        string session_token "JWT hoặc session cookie"
        datetime created_at
        datetime expires_at
    }
```

---

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

---

# Enhance ERD — Thêm Sign in with Google với account linking và CSRF state

Đây là ERD **sau khi** enhance được áp dụng lên base. `USER` và `SESSION` giữ nguyên. So với base, có 2 nhóm entity mới, ứng trực tiếp với yêu cầu trong đề bài:

- `GOOGLE_IDENTITY` (mới) — liên kết 1 user với 1 Google account (`google_sub`, `hosted_domain` để kiểm tra domain công ty), cho phép map với user cũ nếu email đã tồn tại (account linking).
- `OAUTH_STATE` (mới) — lưu tạm `state` param để chống CSRF và `redirect_uri` cần khớp whitelist, tồn tại ngắn hạn trong lúc chờ Google redirect trở lại.

Access token của Google **không** được lưu thành entity nào cả — đúng yêu cầu đề bài "chỉ dùng một lần để lấy profile, không lưu lại".

```mermaid
erDiagram
    USER ||--o{ SESSION : "đăng nhập"
    USER ||--o| GOOGLE_IDENTITY : "có thể liên kết"

    USER {
        string id PK
        string email
        string password_hash "có thể rỗng nếu tạo qua Google lần đầu"
        string name
        datetime created_at
    }
    SESSION {
        string id PK
        string user_id FK
        string session_token "JWT hoặc session cookie"
        datetime created_at
        datetime expires_at
    }
    GOOGLE_IDENTITY {
        string id PK
        string user_id FK
        string google_sub "subject id từ Google"
        string google_email
        string hosted_domain "phải khớp domain công ty"
        datetime linked_at
    }
    OAUTH_STATE {
        string state PK "random, chống CSRF"
        string redirect_uri "phải khớp whitelist"
        datetime created_at
        datetime expires_at
        boolean consumed
    }
```

---

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

---

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
