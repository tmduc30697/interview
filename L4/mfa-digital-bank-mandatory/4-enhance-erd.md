# Enhance ERD — MFA bắt buộc, step-up theo hành vi, OTP có giới hạn, audit thay đổi MFA

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm các entity/trường sau, ứng trực tiếp với từng yêu cầu:

- `MFA_FACTOR` (mới, bắt buộc có ít nhất 1 trước khi `USER.onboarding_completed`) — đáp ứng yêu cầu 1 (bắt buộc enroll trước khi dùng bất kỳ tính năng nào).
- `STEP_UP_POLICY.amount_threshold` và `TRANSACTION.required_step_up`/`step_up_verified_at` — đáp ứng yêu cầu 2 (step-up MFA khi vượt ngưỡng số tiền hoặc thêm beneficiary mới, dù session còn hạn).
- `OTP_CODE` (mới) với `expires_at`, `attempt_count`, `max_attempts`, `status` — đáp ứng yêu cầu 3 (dùng 1 lần, hết hạn nhanh, giới hạn số lần thử sai).
- `OTP_CODE.channel` hỗ trợ chuyển kênh gửi (sms → totp) khi kênh chính lỗi — đáp ứng yêu cầu 4 (fallback rõ ràng khi SMS thất bại/chậm).
- `MFA_CHANGE_AUDIT_LOG` (mới) — đáp ứng yêu cầu 5 (mọi thay đổi liên quan MFA phải tự yêu cầu xác thực MFA hiện tại và gửi cảnh báo qua email).

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    USER ||--o{ BENEFICIARY : "đã lưu"
    USER ||--o{ MFA_FACTOR : enroll
    USER ||--o{ MFA_CHANGE_AUDIT_LOG : "thay đổi MFA"
    ACCOUNT ||--o{ TRANSACTION : "thực hiện"
    BENEFICIARY ||--o{ TRANSACTION : "là người nhận"
    USER ||--o{ OTP_CODE : "yêu cầu xác thực"

    USER {
        string id PK
        string phone_number
        string password_hash
        boolean onboarding_completed "chỉ true khi đã enroll ít nhất 1 MFA_FACTOR"
    }
    ACCOUNT {
        string id PK
        string user_id FK
        decimal balance
    }
    BENEFICIARY {
        string id PK
        string user_id FK
        string name
        string bank_account_number
        datetime added_at
    }
    MFA_FACTOR {
        string id PK
        string user_id FK
        string type "totp|sms"
        string phone_number "cho sms, có thể đổi"
        boolean is_primary
        datetime enrolled_at
    }
    STEP_UP_POLICY {
        string id PK
        decimal amount_threshold
        boolean require_for_new_beneficiary
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string beneficiary_id FK
        string type "transfer|bill_payment"
        decimal amount
        boolean required_step_up
        datetime step_up_verified_at
        datetime created_at
    }
    OTP_CODE {
        string id PK
        string user_id FK
        string purpose "login|step_up_transaction|change_mfa_device"
        string channel "sms|totp"
        string code_hash
        int attempt_count
        int max_attempts
        datetime expires_at
        string status "pending|verified|expired|locked"
    }
    MFA_CHANGE_AUDIT_LOG {
        string id PK
        string user_id FK
        string action "add_device|remove_device|change_phone_number"
        datetime performed_at
        boolean email_alert_sent
    }
```
