# B2B SaaS đa khách hàng tích hợp Okta/Azure AD của từng công ty

**Hệ thống:** Một SaaS quản lý dự án bán cho nhiều công ty khách hàng (multi-tenant), mỗi công ty có hệ thống IdP riêng (Okta, Azure AD, Google Workspace).

**Vai trò của flow:** App đóng vai trò Service Provider, cho phép mỗi tenant tự cấu hình SSO (SAML hoặc OIDC) với IdP của chính họ để nhân viên đăng nhập bằng tài khoản công ty.

**Yêu cầu cụ thể:**
- Mỗi tenant tự nhập metadata IdP (SSO URL, certificate/issuer) qua trang admin riêng, không hard-code cho một provider cụ thể — phải trừu tượng hóa được cả SAML và OIDC sau một interface chung.
- Xác định đúng tenant nào cần redirect tới IdP nào khi user chỉ nhập email (email domain discovery), tránh gửi user tới IdP sai công ty.
- Validate chữ ký SAML response / ID token đúng chuẩn (không tin tưởng field chưa verify), chống replay bằng kiểm tra `InResponseTo`/nonce và thời gian hết hạn của assertion.
- Xử lý case một công ty bắt buộc SSO (disable đăng nhập bằng password) nhưng vẫn cần tài khoản "break-glass" cho admin trong trường hợp IdP sập.
- Tự động provision user mới lần đầu đăng nhập qua SSO (JIT provisioning) và deprovision/khóa account khi IdP báo user bị xóa (SCIM hoặc webhook), không để tài khoản "mồ côi" còn quyền truy cập.
