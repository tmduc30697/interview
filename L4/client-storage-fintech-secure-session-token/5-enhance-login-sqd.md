# Enhance sequence — Login (access token chỉ ở memory, refresh token là cookie httpOnly)

Đây là **enhance**, cùng flow "login" như ở base nhưng thay đổi triệt để nơi lưu 2 loại token. Access token ngắn hạn chỉ giữ trong biến JS ở bộ nhớ (không phải localStorage/sessionStorage) để giảm bề mặt tấn công XSS. Refresh token không còn do JS lưu, mà server set thẳng vào cookie httpOnly + Secure + SameSite, JS không đọc trực tiếp được. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant API as Auth API
    participant Mem as ACCESS_TOKEN_MEMORY (biến JS, RAM)
    participant Cookie as REFRESH_TOKEN_COOKIE (httpOnly, Secure, SameSite)
    participant DB as SERVER_SESSION (server)

    User->>App: Nhập email/mật khẩu, đăng nhập
    App->>API: POST credentials
    API->>DB: Tạo SERVER_SESSION mới, ghi refresh_token_hash
    API-->>App: Trả về access_token trong response body, đồng thời Set-Cookie refresh_token (httpOnly, Secure, SameSite=Strict)
    App->>Mem: Giữ access_token trong biến JS, không ghi vào localStorage/sessionStorage
    Note over Cookie: JS không đọc được nội dung cookie này, trình duyệt tự đính kèm khi gọi API refresh
    Note over Mem: Vì chỉ ở RAM, nếu bị XSS chèn script độc hại thì script cũng không tìm thấy token trong localStorage như base, giảm hẳn bề mặt tấn công so với việc lưu localStorage mà mọi script kể cả script bên thứ ba compromised đều đọc được
    App-->>User: Chuyển vào dashboard
```
