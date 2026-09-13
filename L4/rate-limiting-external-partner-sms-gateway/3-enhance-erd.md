# Enhance ERD — sau khi có throttle outbound tập trung

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `SMS_REQUEST` bổ sung `priority` và `expires_at` — cơ sở để phân biệt OTP (ưu tiên cao, TTL ngắn) với marketing (ưu tiên thấp, TTL dài hơn).
- `OUTBOUND_THROTTLE_QUEUE` (mới) — hàng đợi có TTL cho request bị throttle chờ tới lượt, expire thay vì chờ vô thời hạn.
- `PARTNER_RATE_LIMIT_CONFIG` + `PARTNER_USAGE_WINDOW` (mới) — giới hạn tổng (global) của đối tác và usage thực tế theo từng window gần thời gian thực.
- `PARTNER_BACKOFF_STATE` (mới) — trạng thái backoff tăng dần khi đối tác trả lỗi rate limit.
- `USAGE_ALERT` (mới) — cảnh báo sớm khi usage tiến gần ngưỡng.

```mermaid
erDiagram
    INTERNAL_SERVICE ||--o{ SMS_REQUEST : creates
    SMS_REQUEST ||--o| OUTBOUND_THROTTLE_QUEUE : "waits in (nếu bị throttle)"
    PARTNER_RATE_LIMIT_CONFIG ||--o{ PARTNER_USAGE_WINDOW : tracks
    PARTNER_RATE_LIMIT_CONFIG ||--|| PARTNER_BACKOFF_STATE : "governs backoff for"
    PARTNER_USAGE_WINDOW ||--o{ USAGE_ALERT : "may trigger"

    INTERNAL_SERVICE {
        string id PK
        string name "otp_login | order_notification | marketing"
    }
    SMS_REQUEST {
        string id PK
        string service_id FK
        string message_type "otp | order_notification | marketing"
        string priority "high | normal | low"
        string recipient_phone
        string content
        string status "pending | queued | sent | failed | expired"
        datetime created_at
        datetime expires_at
        datetime sent_at
    }
    OUTBOUND_THROTTLE_QUEUE {
        string id PK
        string sms_request_id FK
        string priority "high | normal | low"
        datetime enqueued_at
        datetime expires_at
        string dequeue_status "queued | dequeued | expired"
    }
    PARTNER_RATE_LIMIT_CONFIG {
        string id PK
        string partner_name
        int max_requests_per_second
    }
    PARTNER_USAGE_WINDOW {
        string id PK
        string rate_limit_config_id FK
        datetime window_start
        datetime window_end
        int request_count
        decimal utilization_pct
    }
    PARTNER_BACKOFF_STATE {
        string id PK
        string rate_limit_config_id FK
        int consecutive_rate_limit_errors
        int current_backoff_seconds
        datetime next_retry_allowed_at
    }
    USAGE_ALERT {
        string id PK
        string usage_window_id FK
        decimal threshold_pct
        datetime triggered_at
        string status "open | resolved"
    }
```
