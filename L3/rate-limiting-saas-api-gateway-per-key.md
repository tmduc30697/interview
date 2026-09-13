# API gateway public cho SaaS đa khách hàng (per-API-key rate limit)

**Hệ thống:** SaaS cung cấp public API cho khách hàng dùng qua API key, cần giới hạn số request theo từng gói dịch vụ (free/pro/enterprise).

**Vai trò của flow:** Rate limiting đảm bảo công bằng tài nguyên giữa các khách hàng, tránh một khách hàng dùng quá nhiều làm ảnh hưởng khách khác, đồng thời đúng với gói dịch vụ họ trả tiền.

**Yêu cầu cụ thể:**
- Mỗi API key có giới hạn request/giây và request/tháng riêng theo gói, và response khi vượt limit phải trả đúng chuẩn (HTTP 429 kèm header Retry-After, X-RateLimit-Remaining).
- Dùng thuật toán rate limit cụ thể (ví dụ token bucket hoặc sliding window log) để cho phép burst ngắn hợp lý mà không cho phép lạm dụng liên tục vượt giới hạn trung bình.
- Rate limit counter phải chính xác trong môi trường nhiều instance API gateway chạy song song (không được để mỗi instance tự đếm riêng dẫn tới tổng vượt xa giới hạn thực).
- Khi Redis/coordinator lưu counter bị chậm/down tạm thời, phải có chiến lược rõ ràng (fail-open cho phép qua tạm hay fail-closed chặn tạm) và giải thích lý do chọn theo bối cảnh SaaS đó.
- Có dashboard cho khách hàng tự xem usage hiện tại so với limit gói của họ, để họ tự chủ động nâng cấp gói trước khi bị chặn.

---

# Base ERD — API gateway forward request, log lại nhưng chưa giới hạn

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có rate limit theo API key. Đề bài nói SaaS có khách hàng dùng API key theo gói dịch vụ (free/pro/enterprise) — nên base cần đủ: khách hàng, API key, và log các lần gọi API (phục vụ billing/analytics cơ bản), nhưng chưa có giới hạn/counter/thuật toán throttle nào, chưa có cấu hình theo gói, chưa có dashboard usage. Những entity đó là phần enhance.

```mermaid
erDiagram
    CUSTOMER ||--o{ API_KEY : owns
    API_KEY ||--o{ API_REQUEST_LOG : "makes calls logged as"

    CUSTOMER {
        string id PK
        string name
        string plan "free | pro | enterprise"
        datetime created_at
    }
    API_KEY {
        string id PK
        string customer_id FK
        string key_value
        string status "active | revoked"
        datetime created_at
    }
    API_REQUEST_LOG {
        string id PK
        string api_key_id FK
        string endpoint
        int response_status
        datetime requested_at
    }
```

---

# Base sequence — Gọi API (forward thẳng, không giới hạn)

Đây là **base**, flow "Gọi API" ở trạng thái hiện tại — gateway chỉ xác thực API key còn hiệu lực rồi forward thẳng tới backend, có log lại request nhưng không đối chiếu với bất kỳ giới hạn nào. Flow này liên quan mật thiết tới enhance vì toàn bộ vấn đề của đề bài (một khách hàng dùng quá nhiều ảnh hưởng khách khác, không đúng với gói đã trả tiền) đều bắt nguồn từ việc thiếu bước kiểm tra giới hạn ở đây.

```mermaid
sequenceDiagram
    actor CustA as Khách hàng A (free plan)
    actor CustB as Khách hàng B (pro plan)
    participant GW as API Gateway (nhiều instance)
    participant Backend as Backend Service
    participant Log as API_REQUEST_LOG

    par Nhiều khách hàng gọi cùng lúc, không phân biệt gói
        CustA->>GW: GET /api/resource (kèm API key)
        GW->>GW: Xác thực API key còn active
        GW->>Backend: Forward request
        Backend-->>GW: Response
        GW->>Log: Ghi API_REQUEST_LOG
        GW-->>CustA: Response
    and
        CustB->>GW: Gọi API liên tục với tốc độ rất cao
        GW->>GW: Xác thực API key còn active
        GW->>Backend: Forward request (không giới hạn tốc độ)
        Backend-->>GW: Response
        GW->>Log: Ghi API_REQUEST_LOG
        GW-->>CustB: Response
    end

    Note over Backend: Khách hàng B dù chỉ trả tiền gói free vẫn có thể gọi vô hạn, chiếm hết tài nguyên backend làm chậm khách hàng A
```

---

# Enhance ERD — sau khi có rate limit per-API-key, phân tán, theo gói

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `PLAN_LIMIT` (mới) — giới hạn request/giây và request/tháng cụ thể theo từng gói.
- `RATE_LIMIT_BUCKET` (mới) — trạng thái token bucket theo từng API key, lưu tập trung (Redis) để chính xác dù nhiều instance gateway chạy song song.
- `MONTHLY_USAGE_COUNTER` (mới) — đếm riêng theo tháng, độc lập với bucket giây, phục vụ cả giới hạn tháng lẫn dashboard usage.
- `COORDINATOR_HEALTH_STATE` (mới) — trạng thái Redis/coordinator và chiến lược fail-open/fail-closed khi nó chậm/down.

```mermaid
erDiagram
    CUSTOMER ||--o{ API_KEY : owns
    API_KEY ||--o{ API_REQUEST_LOG : "makes calls logged as"
    CUSTOMER }o--|| PLAN_LIMIT : "plan matches"
    API_KEY ||--|| RATE_LIMIT_BUCKET : "throttled per second via"
    API_KEY ||--o{ MONTHLY_USAGE_COUNTER : "tracked monthly by"
    RATE_LIMIT_BUCKET }o--|| COORDINATOR_HEALTH_STATE : "checked against"

    CUSTOMER {
        string id PK
        string name
        string plan "free | pro | enterprise"
        datetime created_at
    }
    API_KEY {
        string id PK
        string customer_id FK
        string key_value
        string status "active | revoked"
        datetime created_at
    }
    API_REQUEST_LOG {
        string id PK
        string api_key_id FK
        string endpoint
        int response_status
        datetime requested_at
    }
    PLAN_LIMIT {
        string id PK
        string plan "free | pro | enterprise"
        int requests_per_second
        int requests_per_month
    }
    RATE_LIMIT_BUCKET {
        string id PK
        string api_key_id FK
        string algorithm "token_bucket"
        decimal tokens_remaining
        int bucket_capacity
        datetime last_refill_at
    }
    MONTHLY_USAGE_COUNTER {
        string id PK
        string api_key_id FK
        string month "YYYY-MM"
        int request_count
    }
    COORDINATOR_HEALTH_STATE {
        string id PK
        string coordinator_name "redis_rate_limit_cluster"
        string status "healthy | degraded | down"
        string fallback_strategy "fail_open | fail_closed"
        datetime updated_at
    }
```

---

# Enhance sequence — Gọi API (token bucket dùng chung qua Redis, đúng chuẩn 429)

Đây là **enhance** của flow "Gọi API" đã có ở base. So với base, gateway không còn forward thẳng — mọi instance gateway cùng đọc/ghi chung 1 `RATE_LIMIT_BUCKET` trong Redis theo thuật toán token bucket (cho phép burst ngắn hợp lý) và kiểm tra thêm `MONTHLY_USAGE_COUNTER`, trả đúng chuẩn HTTP 429 kèm `Retry-After`/`X-RateLimit-Remaining` khi vượt (đáp ứng yêu cầu 1, 2 và 3 của đề bài).

```mermaid
sequenceDiagram
    actor CustA as Khách hàng A
    participant GW1 as API Gateway (instance 1)
    participant GW2 as API Gateway (instance 2)
    participant Redis as Redis (RATE_LIMIT_BUCKET dùng chung)
    participant Monthly as MONTHLY_USAGE_COUNTER
    participant Backend as Backend Service

    par 2 request gần như đồng thời, rơi vào 2 instance khác nhau
        CustA->>GW1: GET /api/resource
        GW1->>Redis: Lấy token từ bucket của API key (atomic decrement)
    and
        CustA->>GW2: GET /api/resource
        GW2->>Redis: Lấy token từ bucket của API key (atomic decrement)
    end

    Note over Redis: Counter tập trung ở Redis nên tổng số token bị trừ chính xác dù 2 instance xử lý song song, không đếm riêng lẻ từng instance

    alt Còn token trong bucket và chưa vượt requests_per_month
        Redis-->>GW1: Cho phép, trả tokens_remaining
        GW1->>Monthly: Tăng request_count trong tháng
        GW1->>Backend: Forward request
        Backend-->>GW1: Response
        GW1-->>CustA: 200 OK, header X-RateLimit-Remaining=tokens_remaining
    else Hết token trong bucket (vượt burst cho phép)
        Redis-->>GW2: Từ chối, tính thời gian tới lần refill token tiếp theo
        GW2-->>CustA: 429 Too Many Requests, header Retry-After, X-RateLimit-Remaining=0
    else Đã vượt requests_per_month dù còn token giây
        Monthly-->>GW1: Đã vượt hạn mức tháng
        GW1-->>CustA: 429 Too Many Requests, header Retry-After tới đầu tháng sau
    end
```

---

# Enhance sequence — Redis/coordinator chậm hoặc down, fail-open có kiểm soát

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không phụ thuộc coordinator tập trung nên không có kịch bản nó down). Đáp ứng yêu cầu 4 của đề bài: khi Redis/coordinator lưu counter bị chậm/down tạm thời, phải có chiến lược rõ ràng thay vì hành vi không xác định.

```mermaid
sequenceDiagram
    actor Customer as Khách hàng
    participant GW as API Gateway
    participant Redis as Redis (RATE_LIMIT_BUCKET)
    participant Health as COORDINATOR_HEALTH_STATE
    participant Backend as Backend Service

    Customer->>GW: GET /api/resource
    GW->>Redis: Lấy token từ bucket
    Redis--xGW: Timeout / connection error

    GW->>Health: Cập nhật status=degraded, tăng failure streak

    alt Chọn chiến lược fail-open (mặc định cho SaaS này)
        Note over GW,Health: Ưu tiên availability hơn chặn nhầm khách hàng hợp lệ trong lúc coordinator down ngắn hạn, vì đa số request vẫn nằm trong hạn mức bình thường
        GW->>Backend: Forward request tạm thời, bỏ qua kiểm tra rate limit
        Backend-->>GW: Response
        GW-->>Customer: 200 OK, header X-RateLimit-Remaining=unknown
        GW->>GW: Ghi lại request này vào hàng đợi để đối soát lại usage khi Redis phục hồi
    else Chọn chiến lược fail-closed (khi cần bảo vệ nghiêm ngặt hơn, ví dụ nghi ngờ đang bị tấn công)
        GW-->>Customer: 503 Service Unavailable, không forward tới backend
        Note over GW: Chấp nhận chặn nhầm 1 số khách hàng hợp lệ trong thời gian ngắn để tránh backend bị tràn khi mất kiểm soát rate limit hoàn toàn
    end

    loop Định kỳ kiểm tra lại
        GW->>Redis: Thử kết nối lại
        Redis-->>GW: Phục hồi
        GW->>Health: Cập nhật status=healthy
    end
```

---

# Enhance sequence — Dashboard khách hàng tự xem usage so với limit gói

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có nơi nào để khách hàng xem usage vì chưa có khái niệm limit). Đáp ứng yêu cầu 5 của đề bài: khách hàng tự xem usage hiện tại so với limit gói của họ, chủ động nâng cấp trước khi bị chặn.

```mermaid
sequenceDiagram
    actor Customer as Khách hàng
    participant Portal as Customer Portal
    participant API as Usage API
    participant Bucket as RATE_LIMIT_BUCKET
    participant Monthly as MONTHLY_USAGE_COUNTER
    participant Plan as PLAN_LIMIT

    Customer->>Portal: Mở trang "Usage & Limits"
    Portal->>API: GET /usage?api_key_id=...

    API->>Plan: Đọc requests_per_second, requests_per_month theo plan hiện tại
    API->>Bucket: Đọc tokens_remaining, bucket_capacity gần thời gian thực
    API->>Monthly: Đọc request_count tháng hiện tại

    API-->>Portal: Trả về usage hiện tại vs limit gói
    Portal-->>Customer: Hiển thị biểu đồ req/giây gần đây, và thanh tiến độ request_count/requests_per_month

    alt Usage tháng đang tiến gần requests_per_month, ví dụ trên 80%
        Portal-->>Customer: Gợi ý nâng cấp gói trước khi bị chặn 429 do vượt hạn mức tháng
    end
```
