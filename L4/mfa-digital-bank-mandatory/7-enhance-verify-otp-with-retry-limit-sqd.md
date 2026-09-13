# Enhance sequence — Verify OTP with retry limit (dùng 1 lần, hết hạn nhanh, giới hạn thử sai)

Đây là **enhance**, flow mới phát sinh từ đề bài mô tả chi tiết vòng đời của 1 mã OTP dùng chung cho cả `login` và `transfer-money` (step-up) — mã chỉ dùng được 1 lần, hết hạn sau thời gian ngắn, và bị tạm khóa nếu nhập sai quá số lần cho phép. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor U as User
    participant Server
    participant DB as Database

    Server->>DB: INSERT OTP_CODE (purpose=step_up_transaction, expires_at=now()+5 phút, max_attempts=3, attempt_count=0, status=pending)
    DB-->>Server: OK
    Server-->>U: Đã gửi mã OTP, hết hạn sau 5 phút

    U->>Server: Nhập mã sai lần 1
    Server->>DB: UPDATE OTP_CODE SET attempt_count=1
    Server-->>U: Sai, còn 2 lần thử

    U->>Server: Nhập mã sai lần 2
    Server->>DB: UPDATE OTP_CODE SET attempt_count=2
    Server-->>U: Sai, còn 1 lần thử

    U->>Server: Nhập mã sai lần 3
    Server->>DB: UPDATE OTP_CODE SET attempt_count=3, status=locked
    Server-->>U: Vượt quá số lần thử cho phép, mã OTP này bị khóa, phải yêu cầu gửi mã mới sau thời gian chờ

    Note over Server,DB: Kể cả khi nhập đúng, 1 mã OTP chỉ được dùng chính xác 1 lần, sau khi verify thành công status chuyển sang verified và không thể tái sử dụng

    U->>Server: Yêu cầu gửi lại mã mới sau khi mã cũ hết hạn (quá 5 phút không dùng)
    Server->>DB: SELECT OTP_CODE, kiểm tra expires_at < now()
    DB-->>Server: đã hết hạn
    Server->>DB: INSERT OTP_CODE mới, vô hiệu mã cũ
    Server-->>U: Gửi mã OTP mới
```
