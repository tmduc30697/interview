# Sequence Diagram — Enhance: Break-Glass Login

Đây là **enhance**, biến thể của flow "password-login" ở base nhưng nay bị giới hạn chỉ cho tài khoản `is_break_glass=true` sử dụng khi tenant đã bật `sso_enforced` — mọi user thường khác bị chặn đăng nhập bằng password và buộc phải qua SSO, phòng trường hợp IdP của tenant sập hoàn toàn.

```mermaid
sequenceDiagram
    actor Admin as Break-glass Admin
    participant App as SaaS App

    Admin->>App: Nhập email + password
    App->>App: Tra USER, kiểm tra tenant.sso_enforced

    alt user thường, sso_enforced=true
        App-->>Admin: Từ chối, bắt buộc đăng nhập qua SSO
    else user is_break_glass=true
        App->>App: Kiểm tra password_hash
        App-->>Admin: Đăng nhập thành công qua break-glass, ghi log cảnh báo dùng break-glass
    end
```
