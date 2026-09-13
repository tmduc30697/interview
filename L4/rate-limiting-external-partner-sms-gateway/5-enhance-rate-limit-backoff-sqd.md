# Enhance sequence — Backoff khi đối tác trả lỗi rate limit

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không retry có kiểm soát, không có backoff). Đáp ứng yêu cầu 3 của đề bài: khi bị đối tác trả lỗi rate limit dù đã throttle, phải backoff tăng dần thay vì retry ngay lập tức.

```mermaid
sequenceDiagram
    participant Throttle as Outbound Throttle Gateway
    participant Backoff as PARTNER_BACKOFF_STATE
    participant Gateway as SMS Gateway (đối tác)
    participant Queue as OUTBOUND_THROTTLE_QUEUE

    Throttle->>Gateway: Gọi API gửi SMS (trong hạn mức đã tính toán)
    Gateway-->>Throttle: 429 Rate limit exceeded

    Note over Throttle,Gateway: Dù đã throttle theo config, đối tác vẫn có thể trả lỗi do lệch nhịp đo lường

    Throttle->>Backoff: Tăng consecutive_rate_limit_errors
    Backoff->>Backoff: Tính current_backoff_seconds = base * 2^consecutive_errors (exponential)
    Backoff->>Backoff: Cập nhật next_retry_allowed_at = now + current_backoff_seconds

    Throttle->>Queue: Tạm dừng dequeue mọi request tới đối tác này cho tới next_retry_allowed_at

    Note over Queue: Request mới vẫn được nhận và xếp hàng theo priority, chỉ việc gửi ra ngoài bị tạm dừng

    loop Chờ tới next_retry_allowed_at
        Throttle->>Backoff: Kiểm tra đã tới thời điểm cho phép gửi lại chưa
    end

    Throttle->>Gateway: Gọi lại với backoff đã áp dụng

    alt Thành công
        Gateway-->>Throttle: 200 OK
        Throttle->>Backoff: Reset consecutive_rate_limit_errors về 0
    else Vẫn bị rate limit
        Gateway-->>Throttle: 429 Rate limit exceeded
        Throttle->>Backoff: Tiếp tục tăng backoff, không retry ngay lập tức
    end
```
