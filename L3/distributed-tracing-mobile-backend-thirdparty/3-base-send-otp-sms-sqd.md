# Sequence - Base: Gửi SMS OTP xác thực (có retry nhưng không tách span)

Đây là flow **base**: backend đã có sẵn cơ chế retry khi gọi dịch vụ SMS bên thứ ba bị timeout hoặc lỗi, nhưng toàn bộ các lần thử được ghi chồng vào cùng một span, không phân biệt được "một cuộc gọi chậm" với "nhiều lần gọi cộng dồn". Đây là tiền đề cho yêu cầu 3 (mỗi lần retry là 1 span riêng kèm số thứ tự) và yêu cầu 5 (không ghi nội dung OTP thật vào tag) của đề bài.

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant BE as Backend API
    participant SMS as SMS Provider (bên thứ ba)
    participant Tracer as Tracing Backend

    App->>BE: POST /otp/send (phone_number)
    Note over BE: 1 span duy nhất bao toàn bộ logic gửi OTP, kể cả các lần thử lại
    BE->>BE: Sinh mã OTP
    BE->>SMS: Gửi SMS chứa mã OTP (lần 1)
    SMS-->>BE: Timeout, không phản hồi
    BE->>SMS: Gửi lại SMS (lần 2, cùng span với lần 1)
    SMS-->>BE: Timeout
    BE->>SMS: Gửi lại SMS (lần 3, vẫn cùng span)
    SMS-->>BE: 200 OK, đã gửi
    Note over BE: Tag của span có thể vô tình chứa số điện thoại và nội dung mã OTP thật
    BE-->>App: 200 OK, OTP đã gửi
    BE->>Tracer: Gửi 1 span duy nhất, duration tổng của cả 3 lần thử
```
