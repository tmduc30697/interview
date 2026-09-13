# Enhance ERD — Quota theo tenant, bộ đếm chia sẻ, trọng số theo gói dịch vụ

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm mới:

- `TENANT_QUOTA_POLICY` (max_rps, max_concurrent_connections, route_weight theo `plan_tier`) — đáp ứng yêu cầu 1 (giới hạn tài nguyên độc lập với thuật toán chọn instance) và yêu cầu 4 (trọng số route theo gói dịch vụ, không cần tách backend vật lý riêng).
- `RATE_LIMIT_COUNTER` — bộ đếm request/connection theo tenant, đặt trong 1 store chia sẻ (vd Redis) chứ không lưu cục bộ ở từng `GATEWAY_INSTANCE`, đáp ứng yêu cầu 3 (đếm đúng khi có nhiều gateway instance chạy song song).
- `TRAFFIC_SPIKE_EVENT` — ghi nhận mỗi lần phát hiện traffic tăng đột biến của 1 tenant, kèm phân loại (legit_spike/abnormal) và action đã áp dụng, đáp ứng yêu cầu 2 (phân biệt spike hợp lệ với traffic bất thường).
- `BACKEND_INSTANCE.capacity_weight` — dùng cùng `TENANT_QUOTA_POLICY.route_weight` để phân bổ tài nguyên linh hoạt trên cùng 1 cụm backend, không tách instance riêng theo nhóm tenant.

```mermaid
erDiagram
    TENANT ||--o| TENANT_QUOTA_POLICY : "có 1 policy hiện hành"
    TENANT ||--o{ RATE_LIMIT_COUNTER : "có bộ đếm theo từng window"
    TENANT ||--o{ TRAFFIC_SPIKE_EVENT : "có lịch sử spike"
    GATEWAY_INSTANCE ||--o{ REQUEST_LOG : "xử lý"
    BACKEND_INSTANCE ||--o{ REQUEST_LOG : "phục vụ"

    TENANT {
        string id PK
        string name
        string plan_tier "free|standard|premium"
        string api_key
    }
    TENANT_QUOTA_POLICY {
        string id PK
        string tenant_id FK
        int max_rps
        int max_concurrent_connections
        float burst_multiplier "vd 2x cho phép burst tạm thời"
        float route_weight "trọng số ưu tiên theo plan_tier"
        datetime updated_at
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string tenant_id FK
        datetime window_start
        int current_rps "được cộng dồn từ mọi gateway instance"
        int current_connections
    }
    TRAFFIC_SPIKE_EVENT {
        string id PK
        string tenant_id FK
        datetime detected_at
        float spike_ratio "vd 50x baseline"
        string classification "legit_spike|abnormal"
        string action_taken "allow_with_burst|throttle|block"
        string signal_source "vd error_rate thấp + pattern ổn định = legit"
    }
    GATEWAY_INSTANCE {
        string id PK
        string host
        string status "active|down"
    }
    BACKEND_INSTANCE {
        string id PK
        int active_connections
        float capacity_weight
        string status "active|down"
    }
    REQUEST_LOG {
        string id PK
        string tenant_id FK
        string gateway_instance_id FK
        string backend_instance_id FK
        datetime received_at
        int latency_ms
        string result "success|error|timeout|throttled"
    }
```
