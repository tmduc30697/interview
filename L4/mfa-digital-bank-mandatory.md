# Ngân hàng số bắt buộc MFA cho đăng nhập và giao dịch nhạy cảm

**Hệ thống:** Một app ngân hàng số cho phép chuyển tiền, thanh toán hóa đơn, xem sao kê.

**Vai trò của flow:** MFA bắt buộc ngay từ lần đăng nhập đầu tiên (TOTP app + SMS backup), và step-up MFA riêng cho các giao dịch có giá trị lớn hoặc thêm người thụ hưởng mới.

**Yêu cầu cụ thể:**
- Bắt buộc enroll ít nhất một phương thức MFA trước khi được sử dụng bất kỳ tính năng nào của app, không cho phép "bỏ qua để sau".
- Giao dịch vượt một ngưỡng số tiền hoặc thêm beneficiary mới phải yêu cầu xác thực MFA lại dù session đăng nhập còn hạn (step-up theo hành vi, không chỉ theo thời gian).
- Mã OTP (TOTP/SMS) chỉ dùng được một lần, hết hạn sau một khoảng thời gian ngắn, và phải giới hạn số lần thử sai trước khi tạm khóa xác thực.
- Nếu SMS gửi OTP thất bại hoặc chậm, phải có phương án fallback rõ ràng (ví dụ dùng TOTP app) mà không làm người dùng bị kẹt hoàn toàn ngoài tài khoản.
- Mọi thay đổi liên quan đến MFA (thêm/xóa thiết bị, đổi số điện thoại nhận OTP) phải tự nó yêu cầu xác thực MFA hiện tại, và gửi thông báo cảnh báo qua kênh khác (email) ngay khi thay đổi.

---

# Base ERD — App ngân hàng số chỉ đăng nhập bằng password, giao dịch không có xác thực thêm

Đây là **base**: trạng thái app ngân hàng số *trước khi* bắt buộc MFA. Suy luận từ đề bài, base đã có `USER`, `ACCOUNT` (số dư), `BENEFICIARY` (người thụ hưởng đã lưu), và `TRANSACTION` cho chuyển tiền/thanh toán hóa đơn. Đăng nhập chỉ cần password, giao dịch được thực hiện ngay sau khi đăng nhập mà không có bất kỳ xác thực bổ sung nào, bất kể số tiền lớn hay nhỏ, thêm beneficiary mới hay không.

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    USER ||--o{ BENEFICIARY : "đã lưu"
    ACCOUNT ||--o{ TRANSACTION : "thực hiện"
    BENEFICIARY ||--o{ TRANSACTION : "là người nhận"

    USER {
        string id PK
        string phone_number
        string password_hash
    }
    ACCOUNT {
        string id PK
        string user_id FK
        decimal balance
    }
    BENEFICIARY {
        string id PK
        string user_id FK
        string name
        string bank_account_number
        datetime added_at
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string beneficiary_id FK
        string type "transfer|bill_payment"
        decimal amount
        datetime created_at
    }
```

---

# Base sequence — Login (chỉ password, không có MFA)

Đây là **base**, flow đăng nhập hiện tại: chỉ kiểm tra password, không có bước xác thực bổ sung nào, không có khái niệm enroll MFA. Đây là tiền đề cho yêu cầu 1 của đề bài — hiện tại user có thể dùng toàn bộ app chỉ với password.

```mermaid
sequenceDiagram
    actor U as User
    participant Server
    participant DB as Database

    U->>Server: POST /login (phone_number, password)
    Server->>DB: Xác thực password
    DB-->>Server: OK
    Server->>DB: Tạo session
    DB-->>Server: OK
    Server-->>U: Đăng nhập thành công, truy cập được toàn bộ tính năng ngay
```

---

# Base sequence — Transfer money (không có xác thực bổ sung dù số tiền lớn hay thêm beneficiary mới)

Đây là **base**, flow chuyển tiền hiện tại: giao dịch được thực hiện ngay sau khi user xác nhận trong app, không phân biệt số tiền lớn hay nhỏ, không phân biệt chuyển cho beneficiary đã lưu hay beneficiary mới thêm. Đây là tiền đề cho yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (session đang hoạt động)
    participant Server
    participant DB as Database

    U->>Server: Thêm beneficiary mới (tên, số tài khoản)
    Server->>DB: INSERT BENEFICIARY
    DB-->>Server: OK
    Server-->>U: Đã thêm beneficiary

    U->>Server: Chuyển 500,000,000 VND cho beneficiary vừa thêm
    Server->>DB: SELECT ACCOUNT balance
    DB-->>Server: đủ số dư
    Server->>DB: INSERT TRANSACTION, UPDATE ACCOUNT balance
    DB-->>Server: OK
    Server-->>U: Chuyển tiền thành công ngay lập tức

    Note over Server,DB: Không có bước xác thực bổ sung nào dù số tiền rất lớn và beneficiary vừa mới thêm, chỉ cần session đăng nhập còn hạn
```

---

# Enhance ERD — MFA bắt buộc, step-up theo hành vi, OTP có giới hạn, audit thay đổi MFA

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm các entity/trường sau, ứng trực tiếp với từng yêu cầu:

- `MFA_FACTOR` (mới, bắt buộc có ít nhất 1 trước khi `USER.onboarding_completed`) — đáp ứng yêu cầu 1 (bắt buộc enroll trước khi dùng bất kỳ tính năng nào).
- `STEP_UP_POLICY.amount_threshold` và `TRANSACTION.required_step_up`/`step_up_verified_at` — đáp ứng yêu cầu 2 (step-up MFA khi vượt ngưỡng số tiền hoặc thêm beneficiary mới, dù session còn hạn).
- `OTP_CODE` (mới) với `expires_at`, `attempt_count`, `max_attempts`, `status` — đáp ứng yêu cầu 3 (dùng 1 lần, hết hạn nhanh, giới hạn số lần thử sai).
- `OTP_CODE.channel` hỗ trợ chuyển kênh gửi (sms → totp) khi kênh chính lỗi — đáp ứng yêu cầu 4 (fallback rõ ràng khi SMS thất bại/chậm).
- `MFA_CHANGE_AUDIT_LOG` (mới) — đáp ứng yêu cầu 5 (mọi thay đổi liên quan MFA phải tự yêu cầu xác thực MFA hiện tại và gửi cảnh báo qua email).

```mermaid
erDiagram
    USER ||--o{ ACCOUNT : owns
    USER ||--o{ BENEFICIARY : "đã lưu"
    USER ||--o{ MFA_FACTOR : enroll
    USER ||--o{ MFA_CHANGE_AUDIT_LOG : "thay đổi MFA"
    ACCOUNT ||--o{ TRANSACTION : "thực hiện"
    BENEFICIARY ||--o{ TRANSACTION : "là người nhận"
    USER ||--o{ OTP_CODE : "yêu cầu xác thực"

    USER {
        string id PK
        string phone_number
        string password_hash
        boolean onboarding_completed "chỉ true khi đã enroll ít nhất 1 MFA_FACTOR"
    }
    ACCOUNT {
        string id PK
        string user_id FK
        decimal balance
    }
    BENEFICIARY {
        string id PK
        string user_id FK
        string name
        string bank_account_number
        datetime added_at
    }
    MFA_FACTOR {
        string id PK
        string user_id FK
        string type "totp|sms"
        string phone_number "cho sms, có thể đổi"
        boolean is_primary
        datetime enrolled_at
    }
    STEP_UP_POLICY {
        string id PK
        decimal amount_threshold
        boolean require_for_new_beneficiary
    }
    TRANSACTION {
        string id PK
        string account_id FK
        string beneficiary_id FK
        string type "transfer|bill_payment"
        decimal amount
        boolean required_step_up
        datetime step_up_verified_at
        datetime created_at
    }
    OTP_CODE {
        string id PK
        string user_id FK
        string purpose "login|step_up_transaction|change_mfa_device"
        string channel "sms|totp"
        string code_hash
        int attempt_count
        int max_attempts
        datetime expires_at
        string status "pending|verified|expired|locked"
    }
    MFA_CHANGE_AUDIT_LOG {
        string id PK
        string user_id FK
        string action "add_device|remove_device|change_phone_number"
        datetime performed_at
        boolean email_alert_sent
    }
```

---

# Enhance sequence — Login (bắt buộc enroll MFA trước khi dùng bất kỳ tính năng nào)

Đây là **enhance** của flow `login` đã có ở base. So với base (chỉ password, không MFA), nay user mới không được phép "bỏ qua để sau" — phải hoàn tất enroll ít nhất 1 phương thức MFA (TOTP hoặc SMS backup) ngay sau khi đăng nhập lần đầu, trước khi truy cập bất kỳ tính năng nào của app. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (lần đầu đăng nhập)
    participant Server
    participant DB as Database

    U->>Server: POST /login (phone_number, password)
    Server->>DB: Xác thực password
    DB-->>Server: OK
    Server->>DB: SELECT USER.onboarding_completed
    DB-->>Server: false, chưa enroll MFA nào

    Server-->>U: Bắt buộc enroll MFA trước, chặn mọi tính năng khác, không có nút "bỏ qua"

    U->>Server: Chọn enroll TOTP app, quét QR code
    Server->>DB: INSERT MFA_FACTOR (type=totp, is_primary=true)
    DB-->>Server: OK
    Server-->>U: Yêu cầu nhập mã TOTP để xác nhận enroll đúng

    U->>Server: Nhập mã TOTP đúng
    Server-->>U: Gợi ý thêm SMS backup (khuyến nghị, không bắt buộc thêm phương thức thứ 2)
    U->>Server: Thêm số điện thoại nhận SMS backup
    Server->>DB: INSERT MFA_FACTOR (type=sms, is_primary=false)
    Server->>DB: UPDATE USER SET onboarding_completed=true
    DB-->>Server: OK

    Server-->>U: Hoàn tất, được phép truy cập toàn bộ tính năng app

    Note over U,Server: Ở lần đăng nhập sau, user chỉ cần password + xác thực bằng factor đã enroll (TOTP chính, hoặc SMS backup)
```

---

# Enhance sequence — Transfer money (step-up MFA khi vượt ngưỡng hoặc thêm beneficiary mới)

Đây là **enhance** của flow `transfer-money` đã có ở base. So với base (chuyển tiền ngay không cần xác thực thêm), nay giao dịch vượt ngưỡng số tiền hoặc chuyển cho beneficiary mới thêm phải yêu cầu xác thực MFA lại, dù session đăng nhập còn hạn. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (session đang hoạt động, đã login MFA trước đó)
    participant Server
    participant DB as Database

    U->>Server: Thêm beneficiary mới (tên, số tài khoản)
    Server->>DB: INSERT BENEFICIARY
    DB-->>Server: OK
    Server-->>U: Đã thêm beneficiary

    U->>Server: Chuyển 500,000,000 VND cho beneficiary vừa thêm
    Server->>DB: SELECT STEP_UP_POLICY
    DB-->>Server: amount_threshold=50,000,000, require_for_new_beneficiary=true
    Server->>Server: Kiểm tra amount (500tr > ngưỡng) VÀ beneficiary mới thêm trong phiên này
    Note over Server: Cả 2 điều kiện đều kích hoạt step-up, dù session vẫn còn hạn

    Server->>DB: INSERT TRANSACTION (status=pending, required_step_up=true)
    Server-->>U: Yêu cầu xác thực MFA lại để xác nhận giao dịch này

    U->>Server: Nhập mã OTP từ TOTP app
    Server->>DB: Xác thực OTP hợp lệ
    Server->>DB: UPDATE TRANSACTION SET step_up_verified_at=now(), status=completed
    Server->>DB: UPDATE ACCOUNT balance
    DB-->>Server: OK

    Server-->>U: Chuyển tiền thành công

    Note over Server,DB: Nếu giao dịch dưới ngưỡng và gửi cho beneficiary đã lưu từ trước, không cần step-up, chỉ session hợp lệ là đủ
```

---

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

---

# Enhance sequence — SMS OTP fallback to TOTP (không để user bị kẹt ngoài tài khoản)

Đây là **enhance**, flow mới phát sinh từ đề bài — khi gửi OTP qua SMS thất bại hoặc chậm, hệ thống cung cấp phương án fallback rõ ràng sang TOTP app thay vì để user bị kẹt hoàn toàn ngoài tài khoản. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (chỉ chọn SMS làm phương thức chính lúc login)
    participant Server
    participant SMSProvider as Nhà cung cấp SMS
    participant DB as Database

    Server->>DB: INSERT OTP_CODE (channel=sms, purpose=login, expires_at=now()+5 phút)
    Server->>SMSProvider: Gửi mã OTP tới số điện thoại của U
    SMSProvider-->>Server: Lỗi gửi thất bại, hoặc không phản hồi sau 30 giây

    Server->>DB: UPDATE OTP_CODE SET status=expired (do kênh gửi thất bại)
    Server-->>U: "Không gửi được SMS, bạn có thể dùng TOTP app (nếu đã enroll) để xác thực thay thế"

    alt User đã enroll TOTP app trước đó (is_primary hoặc backup)
        Server->>DB: INSERT OTP_CODE mới (channel=totp, purpose=login)
        Server-->>U: Nhập mã từ TOTP app
        U->>Server: Nhập mã TOTP đúng
        Server->>DB: UPDATE OTP_CODE SET status=verified
        Server-->>U: Đăng nhập thành công qua kênh fallback
    else User chưa enroll TOTP, chỉ có SMS
        Server-->>U: Thử gửi lại SMS qua nhà cung cấp dự phòng thứ 2, hoặc hướng dẫn liên hệ hotline xác minh danh tính thủ công
        Note over Server: Không để user hoàn toàn không có cách nào truy cập tài khoản
    end
```

---

# Enhance sequence — Update MFA device (tự yêu cầu MFA hiện tại + cảnh báo email ngay)

Đây là **enhance**, flow mới phát sinh từ đề bài — mọi thay đổi liên quan tới MFA (thêm/xóa thiết bị, đổi số điện thoại nhận OTP) phải tự nó yêu cầu xác thực MFA hiện tại trước khi thực hiện, và gửi cảnh báo qua email ngay khi thay đổi hoàn tất. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor U as User
    participant Server
    participant DB as Database
    participant EmailService as Dịch vụ gửi email

    U->>Server: Yêu cầu đổi số điện thoại nhận SMS OTP
    Server-->>U: Yêu cầu xác thực MFA hiện tại trước khi cho đổi

    U->>Server: Nhập mã OTP từ TOTP app (factor hiện tại, không phải factor sắp bị đổi)
    Server->>DB: Xác thực OTP hợp lệ

    alt Xác thực MFA thành công
        Server->>DB: UPDATE MFA_FACTOR SET phone_number=<số mới> WHERE type=sms, user=U
        Server->>DB: INSERT MFA_CHANGE_AUDIT_LOG (action=change_phone_number, performed_at=now())
        DB-->>Server: OK

        Server->>EmailService: Gửi email cảnh báo ngay tới địa chỉ email đã đăng ký của U
        EmailService-->>U: "Số điện thoại nhận OTP của bạn vừa được đổi lúc <thời điểm>, nếu không phải bạn hãy liên hệ ngay"

        Server->>DB: UPDATE MFA_CHANGE_AUDIT_LOG SET email_alert_sent=true
        Server-->>U: Đổi số điện thoại thành công
    else Xác thực MFA thất bại
        Server-->>U: Từ chối thay đổi, số điện thoại cũ vẫn giữ nguyên
    end
```
