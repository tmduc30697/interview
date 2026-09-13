# Enhance ERD — sau khi có retry phân loại lỗi + circuit breaker + fallback

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `RETRY_POLICY` (mới) — phân loại rõ lỗi nào được retry (timeout/5xx/429), lỗi nào không (user_error), giới hạn số lần và cách tính backoff (tôn trọng `retry-after` cho 429).
- `CIRCUIT_BREAKER_STATE` + `FALLBACK_PROVIDER_CONFIG` (mới) — mở breaker khi tỉ lệ lỗi vượt ngưỡng, chuyển sang provider dự phòng hoặc trả lỗi thân thiện.
- `RETRY_COST_METRIC` + `FALLBACK_METRIC` (mới) — đo latency/chi phí phát sinh do retry và tỉ lệ phải fallback.

```mermaid
erDiagram
    CHAT_REQUEST ||--o{ PROVIDER_CALL_ATTEMPT : "calls provider via"
    PROVIDER_CALL_ATTEMPT ||--|| RETRY_POLICY : "governed by"
    CHAT_REQUEST ||--o| RETRY_COST_METRIC : measures
    CIRCUIT_BREAKER_STATE ||--o| FALLBACK_PROVIDER_CONFIG : "falls back via"
    CIRCUIT_BREAKER_STATE ||--o{ FALLBACK_METRIC : measured

    CHAT_REQUEST {
        string id PK
        string user_id
        string prompt
        string status "pending | success | failed"
        string provider_used "primary | fallback"
        int token_count
        decimal cost
    }
    PROVIDER_CALL_ATTEMPT {
        string id PK
        string chat_request_id FK
        int attempt_number
        int response_status
        string error_type "timeout | 5xx | 429 | user_error"
        int retry_after_seconds "từ header nếu có"
        datetime attempted_at
    }
    RETRY_POLICY {
        string id PK
        string error_type "timeout | 5xx | 429 | user_error"
        boolean retryable
        int max_attempts
        string backoff_strategy "fixed_with_backoff | respect_retry_after"
    }
    CIRCUIT_BREAKER_STATE {
        string id PK
        string provider_name
        string state "closed | open | half_open"
        decimal error_rate_threshold
        datetime opened_at
    }
    FALLBACK_PROVIDER_CONFIG {
        string id PK
        string primary_provider
        string fallback_provider
        string activation_condition "breaker_open"
    }
    RETRY_COST_METRIC {
        string id PK
        string chat_request_id FK
        int retry_count
        int added_latency_ms
        decimal added_cost
    }
    FALLBACK_METRIC {
        string id PK
        datetime window_start
        datetime window_end
        decimal fallback_rate
    }
```
