# Chống brute-force ở endpoint đăng nhập

**Hệ thống:** Một web app cần bảo vệ endpoint login khỏi tấn công dò mật khẩu (brute force/credential stuffing).

**Vai trò của flow:** Rate limiting theo nhiều chiều (theo IP, theo username, theo device) để làm chậm/ngăn tấn công dò mật khẩu mà không gây khó cho user hợp lệ gõ sai vài lần.

**Yêu cầu cụ thể:**
- Giới hạn số lần login sai theo username cụ thể (ví dụ 5 lần/15 phút) độc lập với giới hạn theo IP (ví dụ 20 lần/15 phút toàn bộ username khác nhau từ 1 IP) để bắt được cả 2 kiểu tấn công (dò 1 tài khoản nhiều lần, và dò nhiều tài khoản từ 1 nguồn).
- Sau khi vượt ngưỡng, phải có cơ chế tăng dần độ khó (progressive delay/CAPTCHA) trước khi chặn hoàn toàn, tránh chặn cứng ngay gây trải nghiệm xấu cho user thật chỉ gõ nhầm vài lần.
- Rate limit không được dựa hoàn toàn vào IP đơn lẻ vì tấn công thực tế thường dùng nhiều IP (botnet) — phải kết hợp thêm fingerprint thiết bị hoặc pattern hành vi bất thường.
- Login thành công phải reset counter rate limit của username đó, không giữ user hợp lệ trong trạng thái gần bị khóa do vài lần gõ sai trước đó.
- Log và alert khi phát hiện pattern tấn công rõ ràng (số lượng lớn login fail trong thời gian ngắn từ nhiều IP nhắm vào nhiều username) để team security theo dõi real-time.

---

# Base ERD — Login đơn giản, chưa có rate limit

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có rate limiting chống brute-force. Đề bài nói bảo vệ endpoint login của 1 web app — nên base cần đủ: user với credential, và session tạo ra khi login thành công. Chưa có entity nào ghi nhận lịch sử các lần login (thành công/thất bại), counter theo username/IP/device, hay cảnh báo tấn công — những thứ đó là phần enhance.

```mermaid
erDiagram
    USER ||--o| SESSION : has

    USER {
        string id PK
        string username
        string password_hash
        datetime created_at
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
```

---

# Base sequence — Login (không giới hạn số lần thử)

Đây là **base**, flow "Login" ở trạng thái hiện tại — kiểm tra username/password, không đếm số lần sai, không phân biệt nguồn gọi. Flow này liên quan mật thiết tới enhance vì toàn bộ vấn đề của đề bài (dò mật khẩu 1 tài khoản nhiều lần, hoặc dò nhiều tài khoản từ 1 nguồn) đều khai thác đúng việc endpoint này cho thử vô hạn lần.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (hoặc user gõ nhầm)
    participant API as Login API
    participant DB as USER store

    loop Có thể lặp lại vô hạn lần, không bị chặn
        Attacker->>API: POST /login (username, password thử)
        API->>DB: Tìm USER theo username, so khớp password_hash
        alt Sai
            DB-->>API: Không khớp
            API-->>Attacker: "Invalid credentials"
        else Đúng
            DB-->>API: Khớp
            API->>API: Tạo SESSION mới
            API-->>Attacker: Login thành công
        end
    end

    Note over API,DB: Không có counter theo username hay IP, không có delay/captcha, không có log cảnh báo pattern bất thường
```

---

# Enhance ERD — sau khi có rate limit đa chiều chống brute-force

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 nhóm entity mới, ứng trực tiếp với các yêu cầu trong đề bài:

- `LOGIN_ATTEMPT` (mới) — ghi nhận từng lần login kèm username, IP, device fingerprint, kết quả — nền tảng để tính counter theo nhiều chiều.
- `RATE_LIMIT_COUNTER` (mới) — đếm số lần fail theo từng chiều độc lập (`username`, `ip`, `device_fingerprint`), bắt được cả tấn công 1 tài khoản lẫn tấn công rải nhiều tài khoản từ 1 nguồn.
- `CHALLENGE_STATE` (mới) — mức độ thử thách tăng dần (delay/CAPTCHA/block) gắn theo counter, thay vì chặn cứng ngay khi vượt ngưỡng.
- `SECURITY_ALERT` (mới) — cảnh báo khi phát hiện pattern tấn công rõ ràng trên diện rộng.

```mermaid
erDiagram
    USER ||--o| SESSION : has
    USER ||--o{ LOGIN_ATTEMPT : "identified in"
    LOGIN_ATTEMPT }o--o{ RATE_LIMIT_COUNTER : increments
    RATE_LIMIT_COUNTER ||--o| CHALLENGE_STATE : escalates
    LOGIN_ATTEMPT ||--o{ SECURITY_ALERT : "may trigger"

    USER {
        string id PK
        string username
        string password_hash
        datetime created_at
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
    LOGIN_ATTEMPT {
        string id PK
        string username
        string ip_address
        string device_fingerprint
        boolean success
        datetime attempted_at
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string dimension "username | ip | device_fingerprint"
        string key_value
        int fail_count
        datetime window_start
        datetime window_end
    }
    CHALLENGE_STATE {
        string id PK
        string dimension "username | ip | device_fingerprint"
        string key_value
        string level "none | progressive_delay | captcha | blocked"
        datetime updated_at
    }
    SECURITY_ALERT {
        string id PK
        string pattern_type "targeted_brute_force | distributed_credential_stuffing"
        int distinct_ip_count
        int distinct_username_count
        datetime triggered_at
        string status "open | resolved"
    }
```

---

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

---

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

---

# Enhance sequence — Phát hiện pattern tấn công diện rộng, alert real-time

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không log hay phân tích pattern của các lần login fail). Đáp ứng yêu cầu 5 của đề bài: log và alert khi phát hiện số lượng lớn login fail trong thời gian ngắn từ nhiều IP nhắm vào nhiều username, để team security theo dõi real-time.

```mermaid
sequenceDiagram
    participant API as Login API
    participant Attempt as LOGIN_ATTEMPT
    participant Detector as Attack Pattern Detector (job real-time)
    participant Alert as SECURITY_ALERT
    actor SecTeam as Team security

    loop Mỗi lần login fail
        API->>Attempt: Ghi LOGIN_ATTEMPT(success=false, ip, username, device_fingerprint)
    end

    loop Quét liên tục theo cửa sổ thời gian ngắn (ví dụ mỗi phút)
        Detector->>Attempt: Đếm distinct_ip_count và distinct_username_count trong window fail gần nhất

        alt distinct_ip_count và distinct_username_count đều vượt ngưỡng bất thường
            Detector->>Alert: Tạo SECURITY_ALERT(pattern_type=distributed_credential_stuffing, status=open)
            Alert-->>SecTeam: Cảnh báo real-time kèm số liệu IP/username liên quan
        else Chỉ 1 username bị nhắm với fail_count rất cao từ ít IP
            Detector->>Alert: Tạo SECURITY_ALERT(pattern_type=targeted_brute_force, status=open)
            Alert-->>SecTeam: Cảnh báo tài khoản cụ thể đang bị dò
        else Không có pattern bất thường
            Detector->>Detector: Không tạo alert
        end
    end

    SecTeam->>Alert: Xử lý xong, đánh dấu status=resolved
```
