# Enhance sequence — Forgot password (request phase, đã hardening)

Đây là **enhance**, cùng flow "Forgot password" đã có ở base nhưng nay thay đổi theo yêu cầu 1, 3 và 5 của đề bài: response giống hệt nhau dù email tồn tại hay không (chống enumeration), rate-limit theo cả email lẫn IP, và bấm "gửi lại" nhiều lần chỉ token mới nhất còn hiệu lực. So với base: không còn tiết lộ email tồn tại hay không, và request bị chặn khi vượt rate-limit thay vì cho gửi email vô hạn.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant RL as RATE_LIMIT_COUNTER store
    participant DB as USER / PASSWORD_RESET_TOKEN store
    participant Email as Email Gateway

    User->>App: Nhập email, bấm "Quên mật khẩu" (hoặc "Gửi lại link")
    App->>RL: Kiểm tra + tăng counter theo email và theo IP
    alt Vượt rate-limit (email hoặc IP)
        RL-->>App: Rate-limit exceeded
        App-->>User: "Nếu email tồn tại, bạn sẽ nhận được link" (response giống hệt case bình thường, không gửi gì thêm)
    else Trong hạn mức
        RL-->>App: OK
        App->>DB: Tìm USER theo email (không tiết lộ kết quả ra ngoài)
        alt Email tồn tại
            DB-->>App: Tìm thấy user
            App->>DB: Vô hiệu mọi PASSWORD_RESET_TOKEN đang active trước đó của user (status=invalidated)
            App->>DB: Tạo PASSWORD_RESET_TOKEN mới (random mạnh, status=active, expires_at=now+20 phút)
            App->>Email: Gửi link chứa token mới
        else Email không tồn tại
            DB-->>App: Không tìm thấy
            Note over App: Không làm gì thêm, không gửi email
        end
        App-->>User: "Nếu email tồn tại, bạn sẽ nhận được link" (response giống hệt case rate-limit/email không tồn tại)
    end
    Note over App,DB: Nếu user bấm "gửi lại" nhiều lần, mỗi lần lặp lại đúng luồng trên — chỉ token mới nhất còn active
```
