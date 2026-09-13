# ERD - Enhance: Span chuẩn hoá cho external call, client network, retry, alert theo provider

Đây là trạng thái **enhance**, sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi chính:

- `SPAN` có thêm `span_kind` (`internal`/`external`), `attempt_number` (số thứ tự lần retry) và `is_retry` — đáp ứng **yêu cầu 1** (tách span external) và **yêu cầu 3** (mỗi lần retry là 1 span riêng).
- Thêm entity `CLIENT_NETWORK_SEGMENT` gắn với `TRACE`, ghi `client_send_ts`/`server_receive_ts` để tính riêng latency mạng di động trước khi tới backend — đáp ứng **yêu cầu 2**.
- `SPAN` liên kết bắt buộc (không còn optional) tới `THIRD_PARTY_PROVIDER` khi `span_kind = external`, và có field `provider_name` ngay trên span để dễ tổng hợp.
- Thêm entity `PROVIDER_ALERT_RULE` (ngưỡng SLA latency/tỷ lệ lỗi riêng theo từng provider) — đáp ứng **yêu cầu 4**.
- Thêm entity `REDACTION_POLICY` quy định whitelist các tag key được phép ghi cho từng loại span/provider, thay cho `SPAN_TAG` tự do ở base — đáp ứng **yêu cầu 5** (không ghi nội dung OTP/toạ độ thật).

```mermaid
erDiagram
    MOBILE_APP ||--o{ BACKEND_REQUEST : sends
    BACKEND_REQUEST ||--|| TRACE : generates
    TRACE ||--|| CLIENT_NETWORK_SEGMENT : "đo latency mạng client"
    TRACE ||--o{ SPAN : contains
    SPAN ||--o{ SPAN_TAG : has
    SPAN }o--|| THIRD_PARTY_PROVIDER : "calls (bắt buộc khi span_kind=external)"
    THIRD_PARTY_PROVIDER ||--o{ PROVIDER_ALERT_RULE : "có ngưỡng SLA riêng"
    THIRD_PARTY_PROVIDER ||--o{ REDACTION_POLICY : "có whitelist tag riêng"

    MOBILE_APP {
        string device_id
        string app_version
    }
    BACKEND_REQUEST {
        string request_id
        string endpoint
        datetime received_at
    }
    TRACE {
        string trace_id
        string request_id
        datetime start_time
        datetime end_time
    }
    CLIENT_NETWORK_SEGMENT {
        string trace_id
        datetime client_send_ts
        datetime server_receive_ts
        int network_latency_ms
    }
    SPAN {
        string span_id
        string trace_id
        string parent_span_id
        string service_name
        string operation_name
        string span_kind
        string provider_name
        int attempt_number
        boolean is_retry
        datetime start_time
        int duration_ms
    }
    SPAN_TAG {
        string span_id
        string key
        string value
        boolean is_redacted
    }
    THIRD_PARTY_PROVIDER {
        string provider_id
        string provider_name
        string type
    }
    PROVIDER_ALERT_RULE {
        string rule_id
        string provider_id
        string metric
        float threshold
        string severity
    }
    REDACTION_POLICY {
        string policy_id
        string provider_id
        string allowed_tag_keys
    }
```
