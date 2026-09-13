# Enhance ERD — sau khi có circuit breaker + retry an toàn

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `CIRCUIT_BREAKER_STATE` (mới) — trạng thái closed/open/half_open, ngưỡng mở, thời điểm mở.
- `IDEMPOTENCY_KEY` (mới) — gắn với mỗi lần charge, dùng để kiểm tra trước khi quyết định retry cho lỗi không rõ kết quả.
- `RETRY_POLICY` (mới) — phân loại rõ lỗi nào được retry, backoff exponential kèm jitter.
- `BREAKER_METRIC` + `BREAKER_ALERT` (mới) — đo tỉ lệ fail-fast, số lần chuyển trạng thái, cảnh báo khi breaker mở quá lâu.

```mermaid
erDiagram
    ORDER ||--o{ PAYMENT_ATTEMPT : "attempts payment via"
    PAYMENT_ATTEMPT ||--|| IDEMPOTENCY_KEY : "tagged with"
    PAYMENT_ATTEMPT ||--|| RETRY_POLICY : "governed by"
    CIRCUIT_BREAKER_STATE ||--o{ BREAKER_METRIC : measured
    CIRCUIT_BREAKER_STATE ||--o{ BREAKER_ALERT : "may trigger"

    ORDER {
        string id PK
        string status "pending | paid | failed"
    }
    PAYMENT_ATTEMPT {
        string id PK
        string order_id FK
        string gateway_request_id
        string status "success | failed | timeout | unknown"
        string error_type "timeout | 5xx | unknown_ambiguous"
        int attempt_number
        datetime attempted_at
    }
    IDEMPOTENCY_KEY {
        string id PK
        string order_id FK
        string key
        string gateway_response_status "success | failed | unknown"
        datetime created_at
    }
    RETRY_POLICY {
        string id PK
        string error_type "timeout | 5xx | unknown_ambiguous"
        boolean retryable
        int backoff_base_ms
        boolean jitter_enabled
    }
    CIRCUIT_BREAKER_STATE {
        string id PK
        string service_name "payment_gateway"
        string state "closed | open | half_open"
        decimal error_rate_threshold
        int error_rate_window_seconds
        datetime opened_at
        int half_open_test_count
    }
    BREAKER_METRIC {
        string id PK
        string service_name FK
        datetime window_start
        datetime window_end
        int fail_fast_count
        int state_transition_count
    }
    BREAKER_ALERT {
        string id PK
        string service_name FK
        datetime opened_at
        int duration_ms
        datetime alert_triggered_at
    }
```
