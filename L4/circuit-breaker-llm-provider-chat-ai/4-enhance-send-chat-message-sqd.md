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
