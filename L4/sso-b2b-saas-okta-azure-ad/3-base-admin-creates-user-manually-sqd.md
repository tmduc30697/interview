# Sequence Diagram — Base: Admin Creates User Manually

Đây là **base**, flow admin tenant tự tạo tài khoản nhân viên thủ công — tiền đề để so sánh với JIT provisioning ở enhance, nơi tài khoản được tự động tạo ngay lần đầu đăng nhập SSO thay vì admin phải tạo tay từng người.

```mermaid
sequenceDiagram
    actor Admin as Tenant Admin
    participant App as SaaS App

    Admin->>App: Nhập thông tin nhân viên mới (email, role)
    App->>App: Create USER record (status=active, password tạm)
    App-->>Admin: Tài khoản đã tạo
    App->>App: Gửi email mời đặt password cho nhân viên
```
