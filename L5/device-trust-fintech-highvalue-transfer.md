# Ngân hàng số quản lý device trust cho giao dịch chuyển tiền giá trị lớn

**Hệ thống:** Một app ngân hàng số cho phép chuyển tiền, mở tài khoản tiết kiệm, và thực hiện các giao dịch tài chính giá trị lớn.

**Vai trò của flow:** Thiết bị mới phải qua xác minh bổ sung trước khi được thực hiện giao dịch giá trị cao, dù đã đăng nhập thành công bằng mật khẩu/MFA thông thường — tách biệt rõ "trust để đăng nhập xem thông tin" và "trust để thực hiện giao dịch lớn".

**Yêu cầu cụ thể:**
- Thiết bị đăng nhập lần đầu chỉ được cấp trust ở mức xem thông tin cơ bản; muốn chuyển khoản vượt một ngưỡng giá trị phải trải qua một cửa sổ "device maturity" (ví dụ thiết bị phải hoạt động ổn định qua một khoảng thời gian, hoặc hoàn tất xác minh bổ sung) trước khi được nâng lên mức trust cho phép giao dịch lớn.
- Phải chặn được kịch bản kẻ tấn công vừa chiếm được thông tin đăng nhập, đăng nhập trên thiết bị mới rồi cố chuyển tiền lớn ngay trong vài giây — tức là race condition giữa việc thiết lập trust thiết bị mới và việc thực hiện giao dịch giá trị cao gần như tức thời sau đó phải được xử lý an toàn theo hướng mặc định từ chối.
- Cần phân biệt hợp lý giữa "thiết bị thực sự mới lạ" và "thiết bị cũ bị cài lại app/xóa dữ liệu" của chính chủ tài khoản, để tránh làm phiền người dùng hợp lệ mỗi lần họ cài lại app trong khi vẫn giữ được mức bảo vệ cần thiết cho trường hợp thật sự đáng ngờ.
- Khi giao dịch bị chặn vì thiết bị chưa đủ trust, phải có luồng step-up rõ ràng và tức thời (ví dụ xác minh qua kênh độc lập như gọi điện, video call, hoặc sinh trắc học bổ sung) cho các trường hợp hợp lệ nhưng cần gấp, thay vì bắt người dùng chờ hàng giờ/ngày một cách cứng nhắc.
- Mọi thay đổi trust level của thiết bị (nâng lên, hạ xuống, revoke) phải có audit trail không thể chỉnh sửa, và phải cảnh báo ngay qua kênh độc lập (SMS/email) mỗi khi có thiết bị mới đạt được mức trust cho phép giao dịch lớn, để chủ tài khoản phát hiện sớm nếu không phải do chính họ thực hiện.

---

# Base ERD — Ngân hàng số chưa phân biệt mức độ tin cậy thiết bị

Đây là **base**: trạng thái hệ thống ngân hàng số *trước khi* có cơ chế device trust theo tầng. Suy luận từ đề bài, base đã có `USER` sở hữu nhiều `ACCOUNT`, đăng nhập từ các `DEVICE` khác nhau (ghi nhận fingerprint, lần đầu thấy), và mỗi `TRANSACTION` (xem số dư hoặc chuyển tiền) được thực hiện qua 1 thiết bị cụ thể. Base chưa phân biệt thiết bị mới hay cũ, chưa có khái niệm mức trust, nên bất kỳ thiết bị nào đăng nhập thành công (đúng mật khẩu/MFA) đều có toàn quyền thực hiện mọi giao dịch kể cả chuyển tiền giá trị lớn ngay lập tức.

```mermaid
erDiagram
    USER ||--o{ DEVICE : "đăng nhập từ"
    USER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ TRANSACTION : "phát sinh"
    DEVICE ||--o{ TRANSACTION : "thực hiện qua"

    USER {
        string id PK
        string name
        string phone
        string email
    }
    DEVICE {
        string id PK
        string user_id FK
        string device_fingerprint
        datetime first_seen_at
    }
    ACCOUNT {
        string id PK
        string user_id FK
        decimal balance
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string device_id FK
        string type "view_balance | transfer"
        decimal amount
        string status "pending|success|failed"
        datetime created_at
    }
```

---

# Base sequence — Login (không phân biệt mức trust thiết bị)

Đây là **base**, flow đăng nhập ở trạng thái hiện tại: sau khi xác thực đúng mật khẩu/MFA, thiết bị (mới hay cũ) đều được cấp toàn quyền truy cập như nhau, hệ thống chỉ ghi nhận fingerprint thiết bị mà không gán mức trust nào. Đây là nền để so sánh với yêu cầu 1 và 3 của đề bài (cần phân tầng trust và phân biệt thiết bị mới thật sự với thiết bị cài lại app).

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant Auth as Auth Service
    participant DB as Database

    User->>App: Nhập username/password
    App->>Auth: Xác thực password + MFA (OTP/app authenticator)
    Auth-->>App: Xác thực thành công

    App->>DB: INSERT/UPDATE DEVICE (device_fingerprint, first_seen_at nếu chưa có)
    DB-->>App: Ghi nhận thiết bị, không gán mức trust nào

    Auth-->>App: Trả access token, toàn quyền như mọi thiết bị khác
    App-->>User: Đăng nhập thành công, có thể xem số dư và chuyển tiền ngay không giới hạn
```

---

# Base sequence — Transfer money (không kiểm tra trust thiết bị, lộ rủi ro kẻ tấn công)

Đây là **base**, flow chuyển tiền ở trạng thái hiện tại: sau khi đăng nhập thành công, thiết bị được phép chuyển tiền với bất kỳ số tiền nào ngay lập tức, không có bước kiểm tra mức độ tin cậy thiết bị. Sơ đồ minh hoạ đúng kịch bản rủi ro nêu ở yêu cầu 2 của đề bài: kẻ tấn công chiếm được thông tin đăng nhập, đăng nhập trên thiết bị mới rồi chuyển tiền lớn thành công chỉ trong vài giây.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (có username/password bị lộ)
    participant App as Mobile App
    participant Auth as Auth Service
    participant Bank as Transfer Service
    participant DB as Database

    Attacker->>App: Đăng nhập trên thiết bị hoàn toàn mới với thông tin đánh cắp
    App->>Auth: Xác thực password + MFA (kẻ tấn công cũng chiếm được OTP qua SIM-swap/phishing)
    Auth-->>App: Xác thực thành công
    App->>DB: Ghi nhận DEVICE mới, không gán mức trust

    Note over Attacker,App: Chỉ vài giây sau khi đăng nhập thành công

    Attacker->>App: Yêu cầu chuyển khoản giá trị lớn ra tài khoản lạ
    App->>Bank: Thực hiện TRANSACTION (type=transfer, amount=lớn)
    Bank->>DB: Không có kiểm tra nào về độ tin cậy thiết bị vừa đăng nhập
    DB-->>Bank: Đủ số dư, không có rào cản nào khác
    Bank-->>App: Chuyển tiền thành công
    App-->>Attacker: Giao dịch hoàn tất, nạn nhân chỉ phát hiện sau khi tiền đã mất
```

---

# Enhance ERD — Device trust theo tầng, cửa sổ maturity, step-up khẩn cấp, audit trail và cảnh báo độc lập

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `DEVICE` được bổ sung nhiều trường theo dõi mức trust, và có 3 entity mới, ứng trực tiếp với các yêu cầu:

- `DEVICE.trust_level`, `device_binding_key`, `maturity_window_ends_at` — đáp ứng yêu cầu 1 (thiết bị mới chỉ ở `basic_view`, phải qua cửa sổ maturity mới lên `high_value_transfer`) và yêu cầu 3 (`device_binding_key` là khoá bền vững lưu ở secure storage của hệ điều hành, dùng để nhận diện thiết bị cũ cài lại app thay vì coi là thiết bị hoàn toàn mới).
- `TRUST_LEVEL_CHANGE_LOG` (mới) — ghi lại mọi lần nâng/hạ/thu hồi trust, không thể sửa đổi — đáp ứng vế đầu yêu cầu 5 (audit trail).
- `STEPUP_VERIFICATION` (mới) — gắn với `TRANSACTION` bị chặn vì thiết bị chưa đủ trust, cho phép xác minh khẩn cấp qua kênh độc lập để nâng trust ngay, bỏ qua thời gian chờ maturity — đáp ứng yêu cầu 4.
- `INDEPENDENT_ALERT` (mới) — gửi SMS/email độc lập mỗi khi thiết bị đạt mức trust cho giao dịch lớn — đáp ứng vế sau yêu cầu 5.
- `TRANSACTION.blocked_reason` — đáp ứng yêu cầu 2 (mặc định từ chối giao dịch lớn khi trust chưa đủ, kể cả ngay sau khi vừa đăng nhập).

```mermaid
erDiagram
    USER ||--o{ DEVICE : "đăng nhập từ"
    USER ||--o{ ACCOUNT : owns
    ACCOUNT ||--o{ TRANSACTION : "phát sinh"
    DEVICE ||--o{ TRANSACTION : "thực hiện qua"
    DEVICE ||--o{ TRUST_LEVEL_CHANGE_LOG : "có lịch sử thay đổi"
    DEVICE ||--o{ STEPUP_VERIFICATION : "có thể cần xác minh khẩn"
    TRANSACTION ||--o| STEPUP_VERIFICATION : "chờ xác minh để mở khoá"
    DEVICE ||--o{ INDEPENDENT_ALERT : "kích hoạt cảnh báo khi lên trust cao"

    USER {
        string id PK
        string name
        string phone
        string email
    }
    DEVICE {
        string id PK
        string user_id FK
        string device_fingerprint
        string device_binding_key "khoá bền vững trong secure storage, sống sót qua cài lại app nếu chưa xoá dữ liệu hệ thống"
        string trust_level "basic_view | high_value_transfer"
        string status "active | revoked"
        datetime first_seen_at
        datetime maturity_window_ends_at
        datetime trust_upgraded_at
    }
    ACCOUNT {
        string id PK
        string user_id FK
        decimal balance
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string device_id FK
        string type "view_balance | transfer"
        decimal amount
        string status "pending|success|failed|blocked_awaiting_stepup"
        string blocked_reason "device_trust_insufficient"
        datetime created_at
    }
    TRUST_LEVEL_CHANGE_LOG {
        string id PK
        string device_id FK
        string previous_trust_level
        string new_trust_level
        string reason "maturity_window_passed | stepup_verified | manual_revoke | reinstall_recognized"
        datetime changed_at
        boolean immutable "true, append-only"
    }
    STEPUP_VERIFICATION {
        string id PK
        string device_id FK
        string transaction_id FK
        string method "independent_call | video_call | extra_biometric"
        string status "pending | verified | failed"
        datetime requested_at
        datetime verified_at
    }
    INDEPENDENT_ALERT {
        string id PK
        string device_id FK
        string channel "sms | email"
        string content
        datetime sent_at
    }
```

---

# Enhance sequence — Login (phân biệt thiết bị mới thật sự với thiết bị cũ cài lại app)

Đây là **enhance** của flow `login` đã có ở base. So với base (mọi thiết bị đăng nhập thành công đều có toàn quyền như nhau), nay thiết bị mới được gán `trust_level=basic_view` mặc định, còn nếu hệ thống nhận diện được `device_binding_key` khớp với thiết bị đã từng đạt trust cao trước đó (cài lại app, chưa xoá secure storage), trust được khôi phục sau một bước xác minh nhẹ thay vì reset về mức thấp nhất. Đáp ứng yêu cầu 1 và 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Mobile App
    participant Auth as Auth Service
    participant DB as Database
    participant Log as TRUST_LEVEL_CHANGE_LOG

    User->>App: Nhập username/password
    App->>Auth: Xác thực password + MFA
    Auth-->>App: Xác thực thành công

    App->>App: Đọc device_binding_key từ secure storage của hệ điều hành (nếu còn tồn tại)
    App->>DB: Gửi kèm device_fingerprint + device_binding_key (nếu có)

    alt Không có device_binding_key hoặc không khớp bản ghi nào (thiết bị mới thật sự)
        DB->>DB: INSERT DEVICE mới (trust_level=basic_view, maturity_window_ends_at=now+7 ngày)
        DB->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(reason=maturity_window_passed sẽ áp dụng sau, hiện tại new_trust_level=basic_view)
        Auth-->>App: Trả access token, chỉ được xem thông tin cơ bản
    else device_binding_key khớp thiết bị cũ đã từng đạt high_value_transfer (cài lại app)
        DB->>App: Yêu cầu xác minh nhẹ bổ sung (vd MFA lần 2 hoặc sinh trắc học có sẵn trên máy)
        App-->>DB: Xác minh nhẹ thành công
        DB->>DB: Khôi phục DEVICE.trust_level = mức trust trước đó, status=active
        DB->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(reason=reinstall_recognized, new_trust_level=mức đã khôi phục)
        Auth-->>App: Trả access token với trust đã khôi phục, không phải chờ lại maturity window
    end

    App-->>User: Đăng nhập thành công
```

---

# Enhance sequence — Nâng trust thiết bị sau cửa sổ maturity, cảnh báo qua kênh độc lập

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Một job định kỳ kiểm tra các thiết bị đã hoạt động ổn định qua đủ thời gian maturity window để nâng trust lên `high_value_transfer`, ghi audit trail bất biến và gửi cảnh báo ngay qua kênh độc lập (SMS/email) cho chủ tài khoản. Đáp ứng yêu cầu 1 (cửa sổ maturity) và yêu cầu 5 (audit trail + cảnh báo độc lập) của đề bài.

```mermaid
sequenceDiagram
    participant Job as Device Maturity Job (định kỳ)
    participant DB as Database
    participant Log as TRUST_LEVEL_CHANGE_LOG
    participant Alert as Independent Alert Service
    actor User

    Job->>DB: SELECT DEVICE WHERE trust_level=basic_view AND maturity_window_ends_at <= now()
    DB-->>Job: Danh sách thiết bị đủ điều kiện xét nâng trust

    loop Với mỗi thiết bị đủ điều kiện
        Job->>DB: Kiểm tra thiết bị có hoạt động ổn định trong suốt cửa sổ (không đổi fingerprint bất thường, không có báo cáo gian lận)
        alt Hoạt động ổn định, không có tín hiệu bất thường
            Job->>DB: UPDATE DEVICE SET trust_level=high_value_transfer, trust_upgraded_at=now()
            Job->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(previous=basic_view, new=high_value_transfer, reason=maturity_window_passed)
            Job->>Alert: Gửi cảnh báo độc lập cho chủ tài khoản
            Alert->>Alert: Gửi SMS/email "Thiết bị {tên} vừa được cấp quyền chuyển tiền giá trị lớn, không phải bạn hãy báo ngay"
            Alert-->>User: Nhận cảnh báo qua kênh độc lập, không phụ thuộc vào app đang đăng nhập
        else Có tín hiệu bất thường trong cửa sổ maturity
            Job->>DB: Reset maturity_window_ends_at, giữ nguyên trust_level=basic_view, chờ thêm 1 chu kỳ
            Job->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(reason=maturity_window_passed nhưng bị từ chối do bất thường, new_trust_level giữ nguyên basic_view)
        end
    end
```

---

# Enhance sequence — Transfer money (kiểm tra trust thiết bị, mặc định từ chối khi chưa đủ, chặn race condition)

Đây là **enhance** của flow `transfer-money` đã có ở base. So với base (chuyển tiền không kiểm tra gì), nay mọi giao dịch vượt ngưỡng giá trị lớn bắt buộc kiểm tra `DEVICE.trust_level=high_value_transfer` ngay tại thời điểm xử lý giao dịch (không dựa vào trạng thái đã cache lúc login), mặc định từ chối nếu thiết bị chưa đạt mức trust này — kể cả khi vừa đăng nhập xong vài giây trước. Đáp ứng yêu cầu 2 của đề bài, tái hiện đúng kịch bản tấn công ở base nhưng lần này bị chặn.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (thiết bị mới, vừa đăng nhập)
    participant App as Mobile App
    participant Bank as Transfer Service
    participant DB as Database

    Attacker->>App: Đăng nhập thành công trên thiết bị mới (trust_level=basic_view, maturity_window chưa qua)
    Note over Attacker,App: Chỉ vài giây sau khi đăng nhập

    Attacker->>App: Yêu cầu chuyển khoản giá trị lớn ra tài khoản lạ
    App->>Bank: Thực hiện TRANSACTION (type=transfer, amount=lớn)
    Bank->>DB: Đọc trust_level hiện tại của DEVICE ngay tại thời điểm xử lý (không dùng giá trị đã cache lúc login)
    DB-->>Bank: trust_level=basic_view (chưa qua maturity window, chưa có stepup)

    alt amount vượt ngưỡng giá trị lớn AND trust_level != high_value_transfer
        Bank->>DB: UPDATE TRANSACTION SET status=blocked_awaiting_stepup, blocked_reason=device_trust_insufficient
        Bank-->>App: Từ chối giao dịch, mặc định deny
        App-->>Attacker: "Thiết bị chưa đủ tin cậy để chuyển khoản giá trị lớn, cần xác minh bổ sung"
        Note over Bank,DB: Mặc định từ chối ngay cả khi thiết bị vừa đăng nhập thành công bằng đúng mật khẩu/MFA, loại bỏ race condition giữa lúc thiết lập trust và lúc thực hiện giao dịch
    else trust_level=high_value_transfer đã được xác lập từ trước
        Bank->>DB: Trừ/cộng số dư, UPDATE TRANSACTION SET status=success
        Bank-->>App: Chuyển tiền thành công
    end
```

---

# Enhance sequence — Step-up khẩn cấp qua kênh độc lập cho giao dịch gấp hợp lệ

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Khi giao dịch bị chặn vì thiết bị chưa đủ trust nhưng người dùng hợp lệ thực sự cần gấp, họ có thể yêu cầu xác minh bổ sung qua kênh độc lập (gọi điện, video call, sinh trắc học bổ sung) để nâng trust ngay lập tức thay vì chờ hết maturity window. Đáp ứng yêu cầu 4, đồng thời vẫn đảm bảo audit trail và cảnh báo độc lập theo yêu cầu 5.

```mermaid
sequenceDiagram
    actor User as Chủ tài khoản hợp lệ
    participant App as Mobile App
    participant Bank as Transfer Service
    participant Verify as Step-up Verification Service
    participant DB as Database
    participant Log as TRUST_LEVEL_CHANGE_LOG
    participant Alert as Independent Alert Service

    User->>App: Giao dịch bị từ chối do device_trust_insufficient
    App-->>User: Hiển thị lựa chọn "Xác minh khẩn cấp ngay" (gọi điện/video call/sinh trắc học bổ sung)
    User->>App: Chọn xác minh khẩn cấp

    App->>Verify: Tạo STEPUP_VERIFICATION (device_id, transaction_id, method=video_call, status=pending)
    Verify->>User: Kết nối video call với tổng đài viên hoặc yêu cầu sinh trắc học bổ sung ngay trên app
    User->>Verify: Hoàn tất xác minh danh tính (khớp giấy tờ/khuôn mặt/giọng nói)

    alt Xác minh thành công
        Verify->>DB: UPDATE STEPUP_VERIFICATION SET status=verified, verified_at=now()
        Verify->>DB: UPDATE DEVICE SET trust_level=high_value_transfer, trust_upgraded_at=now()
        Verify->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(previous=basic_view, new=high_value_transfer, reason=stepup_verified)
        Verify->>Alert: Gửi cảnh báo độc lập cho chủ tài khoản
        Alert->>Alert: Gửi SMS/email "Thiết bị {tên} vừa được xác minh khẩn cấp và cấp quyền giao dịch lớn, không phải bạn hãy báo ngay"

        App->>Bank: Thử lại giao dịch bị chặn trước đó
        Bank->>DB: Đọc lại trust_level, nay đã là high_value_transfer
        Bank->>DB: Trừ/cộng số dư, UPDATE TRANSACTION SET status=success
        Bank-->>App: Chuyển tiền thành công
        App-->>User: Giao dịch hoàn tất ngay trong phiên xác minh, không phải chờ hàng giờ/ngày
    else Xác minh thất bại hoặc nghi ngờ gian lận
        Verify->>DB: UPDATE STEPUP_VERIFICATION SET status=failed
        Verify-->>App: Từ chối nâng trust, giao dịch tiếp tục bị chặn
        App-->>User: Hướng dẫn liên hệ tổng đài hỗ trợ trực tiếp để xử lý thêm
    end
```
