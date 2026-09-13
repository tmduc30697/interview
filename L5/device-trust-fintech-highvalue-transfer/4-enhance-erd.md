# Enhance ERD — Device trust theo tầng, cửa sổ maturity, step-up khẩn cấp, audit trail và cảnh báo độc lập

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `DEVICE` được bổ sung nhiều trường theo dõi mức trust, và có 3 entity mới, ứng trực tiếp với các yêu cầu:

- `DEVICE.trust_level`, `device_binding_key`, `maturity_window_ends_at` — đáp ứng yêu cầu 1 (thiết bị mới chỉ ở `basic_view`, phải qua cửa sổ maturity mới lên `high_value_transfer`) và yêu cầu 3 (`device_binding_key` là khoá bền vững lưu ở secure storage của hệ điều hành, dùng để nhận diện thiết bị cũ cài lại app thay vì coi là thiết bị hoàn toàn mới).
- `TRUST_LEVEL_CHANGE_LOG` (mới) — ghi lại mọi lần nâng/hạ/thu hồi trust, không thể sửa đổi — đáp ứng vế đầu yêu cầu 5 (audit trail).
- `STEPUP_VERIFICATION` (mới) — gắn với `TRANSACTION` bị chặn vì thiết bị chưa đủ trust, cho phép xác minh khẩn cấp qua kênh độc lập để nâng trust ngay, bỏ qua thời gian chờ maturity — đáp ứng yêu cầu 4.
- `INDEPENDENT_ALERT` (mới) — gửi SMS/email độc lập mỗi khi thiết bị đạt mức trust cho giao dịch lớn — đáp ứng vế sau yêu cầu 5.
- `TRANSACTION.blocked_reason` — đáp ứng yêu cầu 2 (mặc định từ chối giao dịch lớn khi trust chưa đủ, kể cả ngay sau khi vừa đăng nhập).

```mermaid
erDiagram
    USER ||--o{ DEVICE : "đăng nhập từ"
    USER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ TRANSACTION : "phát sinh"
    DEVICE ||--o{ TRANSACTION : "thực hiện qua"
    DEVICE ||--o{ TRUST_LEVEL_CHANGE_LOG : "có lịch sử thay đổi"
    DEVICE ||--o{ STEPUP_VERIFICATION : "có thể cần xác minh khẩn"
    TRANSACTION ||--o| STEPUP_VERIFICATION : "chờ xác minh để mở khoá"
    DEVICE ||--o{ INDEPENDENT_ALERT : "kích hoạt cảnh báo khi lên trust cao"

    USER {
        string id PK
        string name
        string phone
        string email
    }
    DEVICE {
        string id PK
        string user_id FK
        string device_fingerprint
        string device_binding_key "khoá bền vững trong secure storage, sống sót qua cài lại app nếu chưa xoá dữ liệu hệ thống"
        string trust_level "basic_view | high_value_transfer"
        string status "active | revoked"
        datetime first_seen_at
        datetime maturity_window_ends_at
        datetime trust_upgraded_at
    }
    ACCOUNT {
        string id PK
        string user_id FK
        decimal balance
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string device_id FK
        string type "view_balance | transfer"
        decimal amount
        string status "pending|success|failed|blocked_awaiting_stepup"
        string blocked_reason "device_trust_insufficient"
        datetime created_at
    }
    TRUST_LEVEL_CHANGE_LOG {
        string id PK
        string device_id FK
        string previous_trust_level
        string new_trust_level
        string reason "maturity_window_passed | stepup_verified | manual_revoke | reinstall_recognized"
        datetime changed_at
        boolean immutable "true, append-only"
    }
    STEPUP_VERIFICATION {
        string id PK
        string device_id FK
        string transaction_id FK
        string method "independent_call | video_call | extra_biometric"
        string status "pending | verified | failed"
        datetime requested_at
        datetime verified_at
    }
    INDEPENDENT_ALERT {
        string id PK
        string device_id FK
        string channel "sms | email"
        string content
        datetime sent_at
    }
```
