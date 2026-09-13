# Base sequence — Login (MFA chỉ là tùy chọn cá nhân, không có chính sách tổ chức)

Đây là **base**, flow đăng nhập hiện tại: hệ thống kiểm tra email/password, sau đó chỉ hỏi MFA nếu chính user đó đã tự nguyện enroll trước đây — không hề tra cứu chính sách của tổ chức mà user đang truy cập. Đây là tiền đề cho toàn bộ đề bài, vì org admin hiện chưa có cách nào ép MFA theo tổ chức của họ.

```mermaid
sequenceDiagram
    actor U as User
    participant Server
    participant DB as Database

    U->>Server: POST /login (email, password)
    Server->>DB: Xác thực password
    DB-->>Server: OK
    Server->>DB: SELECT MFA_ENROLLMENT WHERE user_id=U

    alt User đã tự enroll MFA trước đó
        DB-->>Server: có 1 MFA_ENROLLMENT (totp)
        Server-->>U: Yêu cầu nhập mã TOTP
        U->>Server: Nhập mã TOTP hợp lệ
        Server->>DB: Tạo SESSION mới
        DB-->>Server: OK
        Server-->>U: Đăng nhập thành công
    else User chưa từng enroll MFA
        DB-->>Server: không có MFA_ENROLLMENT nào
        Server->>DB: Tạo SESSION mới, bỏ qua MFA hoàn toàn
        DB-->>Server: OK
        Server-->>U: Đăng nhập thành công, không hỏi MFA
    end

    Note over Server,DB: Không có khái niệm "tổ chức nào yêu cầu MFA", cũng không phân biệt user đang truy cập ở ngữ cảnh org nào
```
