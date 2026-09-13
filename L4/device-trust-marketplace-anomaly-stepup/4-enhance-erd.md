# Enhance ERD — Tách session dài hạn khỏi hành động nhạy cảm, phát hiện bất thường, step-up và báo cáo session

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `SESSION` thêm các trường theo dõi rủi ro, và có 3 entity mới, ứng trực tiếp với các yêu cầu:

- `SENSITIVE_ACTION_VERIFICATION` (mới) — mọi hành động nhạy cảm (`checkout_payment`, `change_address`, `change_payment_method`) cần 1 bản ghi xác minh riêng dù `SESSION` còn hạn — đáp ứng yêu cầu 1.
- `SESSION_ACTIVITY_LOG` (mới) cùng `SESSION.status`/`flagged_reason` — ghi lại IP/geo/user-agent theo từng request để phát hiện thay đổi bất hợp lý giữa các request liên tiếp, gắn cờ session nghi ngờ nhưng không đăng xuất ngay nếu chỉ đang xem sản phẩm — đáp ứng yêu cầu 2 và 3.
- `SENSITIVE_ACTION_VERIFICATION.trigger_reason=payment_after_recent_change` cùng `ADDRESS.updated_at`/`PAYMENT_METHOD.updated_at` — phát hiện thanh toán diễn ra ngay sau khi vừa đổi thông tin nhạy cảm, buộc xác minh lại trước khi hoàn tất — đáp ứng yêu cầu 4.
- `SESSION_REPORT` (mới) cùng `ORDER.locked_for_review` — người dùng xem lịch sử session và báo cáo session lạ, hệ thống tự động khoá tạm các giao dịch đang treo của session đó — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    USER ||--o{ ADDRESS : has
    USER ||--o{ PAYMENT_METHOD : has
    USER ||--o{ ORDER : places
    ADDRESS ||--o{ ORDER : "giao tới"
    PAYMENT_METHOD ||--o{ ORDER : "thanh toán bằng"
    SESSION ||--o{ SESSION_ACTIVITY_LOG : "ghi mọi request"
    SESSION ||--o{ SENSITIVE_ACTION_VERIFICATION : "yêu cầu xác minh khi"
    SESSION ||--o| SESSION_REPORT : "có thể bị báo cáo"
    ORDER ||--o| SENSITIVE_ACTION_VERIFICATION : "gắn với lần thanh toán"

    USER {
        string id PK
        string email
    }
    SESSION {
        string id PK
        string user_id FK
        boolean remember_me
        string device_name
        string ip_address
        string estimated_city
        string estimated_country
        string user_agent
        string status "active | flagged_suspicious | revoked"
        string flagged_reason "ip_geo_jump | user_agent_changed | reported_by_user"
        datetime created_at
        datetime last_seen_at
        datetime expires_at
    }
    SESSION_ACTIVITY_LOG {
        string id PK
        string session_id FK
        string ip_address
        string estimated_city
        string user_agent
        datetime requested_at
    }
    SENSITIVE_ACTION_VERIFICATION {
        string id PK
        string session_id FK
        string order_id FK
        string action_type "checkout_payment | change_address | change_payment_method"
        string trigger_reason "sensitive_action | session_flagged_suspicious | payment_after_recent_change"
        string status "pending | verified | failed"
        datetime requested_at
        datetime verified_at
    }
    SESSION_REPORT {
        string id PK
        string session_id FK
        string reported_by_user_id FK
        datetime reported_at
        string action_taken "session_revoked_and_orders_locked"
    }
    ADDRESS {
        string id PK
        string user_id FK
        string address_line
        boolean is_default
        datetime updated_at
    }
    PAYMENT_METHOD {
        string id PK
        string user_id FK
        string type
        string last4
        boolean is_default
        datetime updated_at
    }
    ORDER {
        string id PK
        string user_id FK
        string address_id FK
        string payment_method_id FK
        decimal amount
        string status "pending|paid|failed"
        boolean locked_for_review
        string locked_reason
        datetime created_at
    }
```
