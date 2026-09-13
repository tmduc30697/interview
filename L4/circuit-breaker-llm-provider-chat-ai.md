# SaaS gọi LLM/AI provider bên thứ ba cho tính năng chat AI

**Hệ thống:** Một SaaS có tính năng chat AI gọi API của nhà cung cấp mô hình ngôn ngữ bên ngoài để trả lời user.

**Vai trò của flow:** Circuit breaker + retry giúp xử lý khi provider AI bị rate limit, timeout, hoặc quá tải, tránh trải nghiệm user bị treo vô thời hạn hoặc gọi lại làm tốn quota/tiền không cần thiết.

**Yêu cầu cụ thể:**
- Retry chỉ thực hiện với lỗi timeout/5xx và tối đa số lần retry giới hạn (ví dụ 2 lần) với backoff, vì mỗi lần gọi provider AI có chi phí (tính theo token) — retry vô tội vạ gây tốn tiền.
- Khi provider trả lỗi 429 (rate limit), phải đọc header retry-after (nếu có) để backoff đúng thời gian được yêu cầu, không tự đoán.
- Circuit breaker mở khi tỉ lệ lỗi/timeout của provider vượt ngưỡng, và hệ thống phải có fallback rõ ràng khi breaker mở (ví dụ chuyển sang provider dự phòng thứ hai, hoặc trả thông báo lỗi thân thiện cho user thay vì treo loading vô hạn).
- Phải phân biệt và không retry với lỗi do chính request của user sai (ví dụ input vượt quá giới hạn token, prompt bị filter) — retry những lỗi này chỉ tốn thời gian mà không giải quyết được gì.
- Đo lường: latency thêm vào do retry (retry overhead), tỉ lệ request phải fallback sang provider dự phòng, và chi phí phát sinh do retry để đưa vào theo dõi ngân sách AI.

---

# Base ERD — Chat AI trước khi có circuit breaker + retry phân loại lỗi

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có retry/circuit breaker thông minh. Đề bài nói SaaS gọi 1 provider AI bên ngoài, tính phí theo token — nên base cần đủ: request chat và các lần gọi provider, chưa phân loại lỗi. Chưa có entity nào phục vụ retry policy/breaker/fallback provider/đo chi phí retry — những thứ đó là phần enhance.

```mermaid
erDiagram
    CHAT_REQUEST ||--o{ PROVIDER_CALL_ATTEMPT : "calls provider via"

    CHAT_REQUEST {
        string id PK
        string user_id
        string prompt
        string status "pending | success | failed"
        int token_count
        decimal cost
    }
    PROVIDER_CALL_ATTEMPT {
        string id PK
        string chat_request_id FK
        int attempt_number
        int response_status
        string error_type
        datetime attempted_at
    }
```

---

# Base sequence — Send chat message (retry mù, không phân biệt lỗi)

Đây là **base**, flow "Gửi tin nhắn chat AI" ở trạng thái hiện tại — retry mọi loại lỗi như nhau, đoán thời gian chờ, không giới hạn rõ số lần. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng các lỗ hổng ở đây.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat SaaS
    participant Provider as LLM Provider

    User->>App: Gửi tin nhắn
    App->>Provider: Gọi API sinh câu trả lời
    Provider-->>App: Lỗi 400 (prompt bị filter/input vượt giới hạn token)
    App->>Provider: Retry ngay (không phân biệt đây là lỗi do chính request của user)
    Provider-->>App: Vẫn lỗi 400 y hệt
    App->>Provider: Retry lần nữa, đoán delay cố định (không đọc header nào)
    Provider-->>App: Lỗi 429 (rate limit)
    App->>Provider: Tiếp tục retry theo delay đoán, có thể ngắn hơn provider yêu cầu
    Note over App,Provider: Mỗi lần retry đều tốn token/tiền dù lỗi 400 không bao giờ tự hết bằng cách gọi lại
    App-->>User: Sau nhiều lần thử, hiển thị lỗi chung chung hoặc user vẫn đang chờ loading
    Note over App: Không đo được đã tốn bao nhiêu chi phí/latency thêm do retry
```

---

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

---

# Enhance sequence — Send chat message (retry phân loại lỗi, tôn trọng retry-after)

Đây là **enhance**, cùng flow "Send chat message" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1, 2, 4 và 5 của đề bài: chỉ retry timeout/5xx với giới hạn số lần, tôn trọng header `retry-after` cho 429, tuyệt đối không retry lỗi do request của user, và đo chi phí/latency phát sinh do retry.

```mermaid
sequenceDiagram
    actor User
    participant App as Chat SaaS
    participant Policy as RETRY_POLICY store
    participant Provider as LLM Provider
    participant Metric as RETRY_COST_METRIC store

    User->>App: Gửi tin nhắn
    App->>Provider: Gọi API sinh câu trả lời
    Provider-->>App: Kết quả

    alt Lỗi 400 (user_error — input vượt token/prompt bị filter)
        App->>Policy: Kiểm tra error_type=user_error → retryable=false
        App-->>User: Trả lỗi rõ ràng ngay, không retry (vd "Nội dung không hợp lệ, vui lòng chỉnh sửa")
    else Lỗi 429 (rate limit)
        App->>Provider: Đọc header retry-after
        App->>Policy: Kiểm tra max_attempts chưa vượt
        App->>App: Chờ đúng retry_after_seconds (không tự đoán)
        App->>Provider: Retry sau đúng khoảng thời gian yêu cầu
        Provider-->>App: Thành công
        App->>Metric: Ghi retry_count, added_latency_ms, added_cost
        App-->>User: Trả kết quả
    else Lỗi timeout/5xx
        App->>Policy: Kiểm tra retryable=true, max_attempts=2
        loop Tối đa 2 lần, có backoff
            App->>Provider: Retry
            Provider-->>App: Kết quả
        end
        alt Cuối cùng thành công
            App->>Metric: Ghi chi phí/latency phát sinh do retry
            App-->>User: Trả kết quả
        else Vẫn thất bại sau max_attempts
            App-->>User: Chuyển sang flow "Circuit breaker & fallback"
        end
    end
```

---

# Enhance sequence — Circuit breaker & fallback provider

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có breaker/fallback, chỉ retry vô hạn). Đáp ứng yêu cầu thứ 3 và phần còn lại của yêu cầu thứ 5 của đề bài: breaker mở khi tỉ lệ lỗi vượt ngưỡng, fallback rõ ràng sang provider dự phòng hoặc lỗi thân thiện, và đo tỉ lệ fallback.

```mermaid
sequenceDiagram
    participant Breaker as CIRCUIT_BREAKER_STATE (provider chính)
    participant Primary as LLM Provider chính
    participant Fallback as FALLBACK_PROVIDER_CONFIG store
    participant FallbackProvider as LLM Provider dự phòng
    participant Metric as FALLBACK_METRIC store
    actor User

    loop Theo dõi liên tục
        Primary-->>Breaker: Cập nhật tỉ lệ lỗi/timeout
    end

    alt Tỉ lệ lỗi vượt ngưỡng
        Breaker->>Breaker: Chuyển state=open
    end

    User->>Primary: Gửi tin nhắn (đã hết max_attempts retry ở flow "Send chat message", hoặc breaker đang open)
    Primary-->>Breaker: Kiểm tra state
    alt Breaker = open và có cấu hình fallback
        Breaker->>Fallback: Lấy fallback_provider tương ứng
        Fallback-->>FallbackProvider: Route request sang provider dự phòng
        FallbackProvider-->>User: Trả kết quả (từ provider dự phòng, không treo loading vô hạn)
        Breaker->>Metric: Ghi nhận 1 request đã fallback trong window hiện tại
    else Breaker = open nhưng không có fallback khả dụng
        Breaker-->>User: Trả thông báo lỗi thân thiện ("Trợ lý AI đang tạm thời quá tải, vui lòng thử lại sau") thay vì treo vô hạn
    end

    Metric-->>Metric: Tổng hợp fallback_rate theo window để theo dõi ngân sách AI
```
