# ERD - Base: Tracing cơ bản trước khi tách bạch external call

Đây là trạng thái **base**, trước khi áp đề bài. Hệ thống đã có tracing phân tán cơ bản (mỗi request từ mobile app sinh 1 trace, trace chứa nhiều span), nhưng span chưa có cấu trúc chuẩn hoá để phân biệt "gọi ra bên thứ ba" với "xử lý nội bộ", chưa ghi nhận số lần retry, chưa tách latency mạng di động của client, và tag của span là key-value tự do không kiểm soát nội dung nhạy cảm. Đây là nền tảng tối thiểu để đề bài (tách span external, gắn tag provider, retry span, alert riêng theo provider, chống rò rỉ PII) "có ý nghĩa" khi so sánh.

```mermaid
erDiagram
    MOBILE_APP ||--o{ BACKEND_REQUEST : sends
    BACKEND_REQUEST ||--|| TRACE : generates
    TRACE ||--o{ SPAN : contains
    SPAN ||--o{ SPAN_TAG : has
    SPAN }o--o| THIRD_PARTY_PROVIDER : "calls (không bắt buộc, không chuẩn hoá)"

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
    SPAN {
        string span_id
        string trace_id
        string parent_span_id
        string service_name
        string operation_name
        datetime start_time
        int duration_ms
    }
    SPAN_TAG {
        string span_id
        string key
        string value
    }
    THIRD_PARTY_PROVIDER {
        string provider_id
        string provider_name
        string type
    }
```

Lưu ý: `SPAN` ở base không có field `span_kind` (internal/external), không có `attempt_number` cho retry, và `SPAN_TAG` là key-value tự do, không có whitelist hay quy tắc chống ghi nội dung nhạy cảm — đây chính là những khoảng trống mà enhance sẽ lấp.
