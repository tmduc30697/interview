# Sequence Diagram — Enhance: SCIM Deprovision User

Đây là **enhance**, flow hoàn toàn mới xử lý khi IdP báo user bị xóa — hệ thống phải khóa/deprovision tài khoản tương ứng qua SCIM hoặc webhook, tránh tài khoản "mồ côi" còn quyền truy cập sau khi đã bị xóa ở phía IdP.

```mermaid
sequenceDiagram
    participant IdP as IdP của tenant
    participant App as SaaS App (Service Provider)

    IdP->>App: SCIM webhook - user bị xóa/disable ở phía IdP
    App->>App: Tra USER theo email/external_id trong đúng tenant
    App->>App: Mark USER status=deprovisioned, deprovisioned_at=now
    App->>App: Revoke toàn bộ SSO_SESSION đang hoạt động của user
    App-->>IdP: Xác nhận đã deprovision
```
