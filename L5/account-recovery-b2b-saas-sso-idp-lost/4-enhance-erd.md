# Enhance ERD — sau khi có cơ chế khôi phục cấp tổ chức

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `IDP_CONFIG` chuyển từ 1-1 (1 active/org) sang 1-N, thêm `status` (active/retiring/retired/candidate) và khoảng hiệu lực — để hỗ trợ chạy song song 2 IdP trong lúc cutover có kế hoạch.
- `BREAK_GLASS_ACCOUNT` (mới) — tài khoản cấp tổ chức được thiết lập sẵn từ trước, không tạo lúc khẩn cấp, có MFA riêng và theo dõi lần dùng gần nhất.
- `SSO_CHANGE_REQUEST` (mới) — mọi yêu cầu đổi/cấu hình lại IdP (dù chủ động hay khẩn cấp) đều phải đi qua 1 request có xác minh + trạng thái duyệt, thay vì ghi đè trực tiếp như base.
- `OUTAGE_INCIDENT` (mới) — ghi nhận và phân loại sự cố IdP (tạm thời hay thực sự cần chuyển) trước khi cho phép mở `SSO_CHANGE_REQUEST` loại khẩn cấp.
- `AUDIT_LOG` (mới) — giám sát chặt việc sử dụng break-glass account.

```mermaid
erDiagram
    ORGANIZATION ||--o{ USER : employs
    ORGANIZATION ||--o{ IDP_CONFIG : "configured with (nhiều, theo status)"
    ORGANIZATION ||--o{ BREAK_GLASS_ACCOUNT : "provisioned for"
    ORGANIZATION ||--o{ SSO_CHANGE_REQUEST : requests
    ORGANIZATION ||--o{ OUTAGE_INCIDENT : experiences
    USER ||--o{ SESSION : creates
    BREAK_GLASS_ACCOUNT ||--o{ AUDIT_LOG : "usage logged in"
    SSO_CHANGE_REQUEST ||--o| OUTAGE_INCIDENT : "may cite"
    SSO_CHANGE_REQUEST ||--o| IDP_CONFIG : "produces/updates"

    ORGANIZATION {
        string id PK
        string name
        string status
    }
    IDP_CONFIG {
        string id PK
        string org_id FK
        string provider_type
        string metadata_url
        string signing_cert
        string status "active | retiring | retired | candidate"
        datetime valid_from
        datetime valid_until
    }
    USER {
        string id PK
        string org_id FK
        string email
        string role
        string external_idp_subject_id
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
    BREAK_GLASS_ACCOUNT {
        string id PK
        string org_id FK
        string credential_hash
        string mfa_secret
        string status
        datetime provisioned_at
        datetime last_used_at
    }
    SSO_CHANGE_REQUEST {
        string id PK
        string org_id FK
        string type "planned_cutover | emergency_recovery"
        string requested_by
        string verification_status
        string approval_status
        datetime created_at
        datetime cutover_start
        datetime cutover_end
    }
    OUTAGE_INCIDENT {
        string id PK
        string org_id FK
        string idp_config_id FK
        datetime detected_at
        int duration_minutes
        string classification "transient | confirmed_outage"
        datetime resolved_at
    }
    AUDIT_LOG {
        string id PK
        string break_glass_account_id FK
        string action
        datetime created_at
    }
```
