# Sequence - Enhance: Gửi SMS OTP (mỗi lần retry 1 span, không ghi nội dung OTP)

Đây là flow **enhance** của `send-otp-sms` (so với base). Khác biệt so với base: mỗi lần gọi SMS provider (kể cả các lần retry) được bọc trong 1 span external riêng, gắn `attempt_number` tăng dần và `is_retry=true` từ lần thứ 2 trở đi, để phân biệt rõ "1 cuộc gọi chậm" với "3 lần gọi cộng dồn". Tag của span chỉ ghi metadata debug (status, duration, error_code), không ghi số điện thoại hay nội dung mã OTP thật. Đáp ứng **yêu cầu 1, 3 và 5** của đề bài.

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant BE as Backend API
    participant SMS as SMS Provider (bên thứ ba)
    participant Tracer as Tracing Backend

    App->>BE: POST /otp/send (phone_number)
    Note over BE: Span internal (span_kind=internal) bao logic sinh OTP
    BE->>BE: Sinh mã OTP
    Note over BE: Span external #1, span_kind=external, provider_name=sms, attempt_number=1
    BE->>SMS: Gửi SMS (lần 1)
    SMS-->>BE: Timeout
    Note over BE: Kết thúc span #1 với status=timeout, không ghi nội dung OTP/SĐT vào tag
    Note over BE: Span external #2, attempt_number=2, is_retry=true
    BE->>SMS: Gửi lại SMS (lần 2)
    SMS-->>BE: Timeout
    Note over BE: Kết thúc span #2 với status=timeout
    Note over BE: Span external #3, attempt_number=3, is_retry=true
    BE->>SMS: Gửi lại SMS (lần 3)
    SMS-->>BE: 200 OK, đã gửi
    Note over BE: Kết thúc span #3 với status=success, chỉ ghi status_code và duration
    BE-->>App: 200 OK, OTP đã gửi
    BE->>Tracer: Gửi 3 span external riêng biệt (attempt 1, 2, 3) cộng 1 span internal
```
