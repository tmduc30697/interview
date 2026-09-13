# ERD — Enhance (sau khi có SSO qua IdP của từng tenant)

Đây là **enhance**: ERD base cộng với các entity phục vụ đúng yêu cầu đề bài — cấu hình IdP trừu tượng hóa SAML/OIDC cho từng tenant, ánh xạ email domain sang tenant, phiên SSO chống replay, cờ break-glass, và JIT/SCIM provisioning. So với base, `USER` nay có `auth_source` và `is_break_glass`, còn việc tạo user không chỉ do admin tạo tay mà còn tự động qua SSO lần đầu.

```mermaid
erDiagram
    TENANT ||--o{ USER : employs
    TENANT ||--|| IDP_CONFIG : "configures"
    TENANT ||--o{ EMAIL_DOMAIN_MAPPING : owns
    USER ||--o{ SSO_SESSION : "creates"

    TENANT {
        string tenant_id PK
        string name
        string status
        boolean sso_enforced
    }

    IDP_CONFIG {
        string idp_config_id PK
        string tenant_id FK
        string protocol
        string sso_url
        string issuer
        string certificate
        string status
    }

    EMAIL_DOMAIN_MAPPING {
        string mapping_id PK
        string tenant_id FK
        string email_domain
    }

    USER {
        string user_id PK
        string tenant_id FK
        string email
        string password_hash
        string role
        string auth_source
        boolean is_break_glass
        string status
        datetime created_at
        datetime deprovisioned_at
    }

    SSO_SESSION {
        string session_id PK
        string user_id FK
        string idp_config_id FK
        string in_response_to
        string nonce
        datetime issued_at
        datetime expires_at
    }
```
