# Enhance sequence — Tăng dần độ khó khi vượt ngưỡng, kết hợp device fingerprint

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm thử thách hay khóa tạm). Đáp ứng yêu cầu 2 và 3 của đề bài: không chặn cứng ngay lập tức, tăng dần progressive delay/CAPTCHA trước khi block hẳn, và không chỉ dựa vào IP đơn lẻ vì tấn công thực tế thường dùng botnet nhiều IP.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (nhiều IP, có thể cùng device fingerprint)
    participant API as Login API
    participant Counter as RATE_LIMIT_COUNTER
    participant Challenge as CHALLENGE_STATE

    Attacker->>API: POST /login (lần thử vượt ngưỡng username hoặc ip)
    API->>Counter: Đọc fail_count đã vượt ngưỡng
    API->>Challenge: Đọc level hiện tại theo dimension=username

    alt level=none, vừa vượt ngưỡng lần đầu
        API->>Challenge: Nâng level=progressive_delay
        API-->>Attacker: Yêu cầu chờ vài giây trước khi thử lại
    else level=progressive_delay, vẫn tiếp tục sai
        API->>Challenge: Nâng level=captcha
        API-->>Attacker: Yêu cầu giải CAPTCHA trước khi thử lại
    else level=captcha, vẫn tiếp tục sai
        API->>Challenge: Nâng level=blocked
        API-->>Attacker: Chặn hoàn toàn theo username trong 1 khoảng thời gian
    end

    Note over API,Challenge: Đồng thời API cũng theo dõi device_fingerprint trùng lặp giữa nhiều IP khác nhau

    alt Nhiều IP khác nhau cùng chung 1 device_fingerprint đang thử cùng username
        API->>Challenge: Nâng level=captcha cho toàn bộ request mang fingerprint đó, bất kể IP nào gửi tới
        Note over API: Không thể lách rate limit chỉ bằng cách đổi IP (botnet) nếu fingerprint không đổi
    end
```
