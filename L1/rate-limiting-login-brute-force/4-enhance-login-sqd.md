# Enhance sequence — Login (kiểm tra rate limit đa chiều, reset khi thành công)

Đây là **enhance** của flow "Login" đã có ở base. So với base, mỗi lần login đều được ghi nhận và đối chiếu với counter theo cả username lẫn IP trước khi xử lý; login thành công reset counter theo username; login sai cả username lẫn IP đều bị tăng độc lập (đáp ứng yêu cầu 1 và 4 của đề bài).

```mermaid
sequenceDiagram
    actor User as User (hợp lệ hoặc tấn công)
    participant API as Login API
    participant Counter as RATE_LIMIT_COUNTER
    participant DB as USER store
    participant Attempt as LOGIN_ATTEMPT

    User->>API: POST /login (username, password, ip, device_fingerprint)

    API->>Counter: Đọc fail_count(dimension=username, key=username) trong 15 phút gần nhất
    API->>Counter: Đọc fail_count(dimension=ip, key=ip_address) trong 15 phút gần nhất

    alt Cả 2 counter đều dưới ngưỡng (username dưới 5, ip dưới 20)
        API->>DB: So khớp password_hash
        API->>Attempt: Ghi LOGIN_ATTEMPT(username, ip, device_fingerprint, success)

        alt Đúng
            API->>Counter: Reset fail_count(dimension=username) về 0
            Note over Counter: Login thành công không giữ user ở trạng thái gần bị khóa vì vài lần gõ sai trước đó
            API->>API: Tạo SESSION mới
            API-->>User: Login thành công
        else Sai
            API->>Counter: Tăng fail_count(dimension=username)
            API->>Counter: Tăng fail_count(dimension=ip)
            API-->>User: "Invalid credentials"
        end
    else Một trong 2 counter đã vượt ngưỡng
        API-->>User: Yêu cầu vượt qua thử thách bổ sung trước khi thử tiếp
        Note over API,User: Chi tiết cơ chế tăng dần độ khó ở flow progressive-challenge riêng
    end
```
