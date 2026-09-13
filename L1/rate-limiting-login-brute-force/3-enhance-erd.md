# Enhance ERD — sau khi có rate limit đa chiều chống brute-force

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `LOGIN_ATTEMPT` (mới) — ghi nhận từng lần login kèm username, IP, device fingerprint, kết quả — nền tảng để tính counter theo nhiều chiều.
- `RATE_LIMIT_COUNTER` (mới) — đếm số lần fail theo từng chiều độc lập (`username`, `ip`, `device_fingerprint`), bắt được cả tấn công 1 tài khoản lẫn tấn công rải nhiều tài khoản từ 1 nguồn.
- `CHALLENGE_STATE` (mới) — mức độ thử thách tăng dần (delay/CAPTCHA/block) gắn theo counter, thay vì chặn cứng ngay khi vượt ngưỡng.
- `SECURITY_ALERT` (mới) — cảnh báo khi phát hiện pattern tấn công rõ ràng trên diện rộng.

```mermaid
erDiagram
    USER ||--o| SESSION : has
    USER ||--o{ LOGIN_ATTEMPT : "identified in"
    LOGIN_ATTEMPT }o--o{ RATE_LIMIT_COUNTER : increments
    RATE_LIMIT_COUNTER ||--o| CHALLENGE_STATE : escalates
    LOGIN_ATTEMPT ||--o{ SECURITY_ALERT : "may trigger"

    USER {
        string id PK
        string username
        string password_hash
        datetime created_at
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
    LOGIN_ATTEMPT {
        string id PK
        string username
        string ip_address
        string device_fingerprint
        boolean success
        datetime attempted_at
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string dimension "username | ip | device_fingerprint"
        string key_value
        int fail_count
        datetime window_start
        datetime window_end
    }
    CHALLENGE_STATE {
        string id PK
        string dimension "username | ip | device_fingerprint"
        string key_value
        string level "none | progressive_delay | captcha | blocked"
        datetime updated_at
    }
    SECURITY_ALERT {
        string id PK
        string pattern_type "targeted_brute_force | distributed_credential_stuffing"
        int distinct_ip_count
        int distinct_username_count
        datetime triggered_at
        string status "open | resolved"
    }
```
