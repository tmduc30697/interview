# Base sequence — Login

Đây là **base**, flow "Đăng nhập bằng email/password" — nền tảng để hiểu tại sao flow quên mật khẩu và các session đang hoạt động (mà enhance sẽ cần thu hồi) lại quan trọng.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant DB as USER store

    User->>App: Nhập email + password
    App->>DB: Tra password_hash theo email
    DB-->>App: Trả về password_hash
    App->>App: So khớp hash
    App->>App: Tạo SESSION mới
    App-->>User: Đăng nhập thành công
```
