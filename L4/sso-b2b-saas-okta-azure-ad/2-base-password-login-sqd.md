# Sequence Diagram — Base: Password Login

Đây là **base**, flow đăng nhập bằng email/password hiện tại của mỗi tenant — tiền đề bắt buộc vì SSO sẽ thay thế phần lớn flow này cho user thường, chỉ còn giữ lại cho tài khoản break-glass ở enhance.

```mermaid
sequenceDiagram
    actor User as Nhân viên tenant
    participant App as SaaS App (Service Provider)

    User->>App: Nhập email + password
    App->>App: Tra USER theo email trong tenant
    App->>App: Kiểm tra password_hash
    alt hợp lệ
        App-->>User: Đăng nhập thành công, tạo session
    else sai
        App-->>User: Từ chối đăng nhập
    end
```
