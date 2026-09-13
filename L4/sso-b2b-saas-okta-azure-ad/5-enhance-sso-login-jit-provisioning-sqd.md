# Sequence Diagram — Enhance: SSO Login JIT Provisioning

Đây là **enhance**, flow hoàn toàn mới thay thế phần lớn base: user chỉ nhập email, hệ thống tra `EMAIL_DOMAIN_MAPPING` để biết redirect tới IdP nào, validate chữ ký/chống replay, rồi tự động tạo `USER` ngay lần đầu đăng nhập (JIT) thay vì admin phải tạo tay như flow "admin-creates-user-manually" ở base.

```mermaid
sequenceDiagram
    actor User as Nhân viên tenant
    participant App as SaaS App (Service Provider)
    participant IdP as IdP của tenant (Okta/Azure AD)

    User->>App: Nhập email
    App->>App: Tra EMAIL_DOMAIN_MAPPING, xác định đúng tenant + IDP_CONFIG
    App->>IdP: Redirect SSO request (SAML/OIDC theo IDP_CONFIG.protocol)
    User->>IdP: Đăng nhập tại IdP của công ty
    IdP-->>App: SAML response / ID token (InResponseTo, nonce, chữ ký)

    App->>App: Validate chữ ký theo certificate của IDP_CONFIG
    App->>App: Kiểm tra InResponseTo/nonce chống replay, kiểm tra assertion chưa hết hạn

    alt user chưa tồn tại
        App->>App: Tạo USER mới (auth_source=sso) - JIT provisioning
    else user đã tồn tại
        App->>App: Cập nhật thông tin nếu cần
    end

    App->>App: Tạo SSO_SESSION (in_response_to, nonce, expires_at)
    App-->>User: Đăng nhập thành công
```
