# Base sequence — SSO Login

Đây là **base**, flow "Đăng nhập qua SSO" — tiền đề trực tiếp cho enhance: chính vì mọi nhân viên chỉ đăng nhập được qua **đúng 1** IdP đang active của tổ chức, nên khi IdP đó hỏng, toàn bộ tổ chức bị khóa cùng lúc (vấn đề gốc mà đề bài yêu cầu giải quyết).

```mermaid
sequenceDiagram
    actor Employee
    participant App as SaaS App (SP)
    participant DB as IDP_CONFIG store
    participant IdP as Org's IdP

    Employee->>App: Truy cập app bằng email công ty
    App->>DB: Tìm IDP_CONFIG active theo org (suy ra từ domain email)
    DB-->>App: Trả về 1 IDP_CONFIG active
    App-->>Employee: Redirect sang IdP để xác thực
    Employee->>IdP: Đăng nhập (username/password/MFA tại IdP)
    IdP-->>App: Trả assertion (SAML/OIDC) đã ký
    App->>App: Verify chữ ký assertion bằng cert trong IDP_CONFIG
    App->>App: Tìm/khớp USER theo external_idp_subject_id
    App->>App: Tạo SESSION mới
    App-->>Employee: Đăng nhập thành công
```
