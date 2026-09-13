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
