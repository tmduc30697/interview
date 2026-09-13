# Marketplace e-commerce phát hiện session bất thường và yêu cầu step-up

**Hệ thống:** Một sàn thương mại điện tử cho phép người dùng lưu thông tin thanh toán và địa chỉ để mua hàng nhanh.

**Vai trò của flow:** Quản lý session "remember me" thông thường cho trải nghiệm mua sắm hằng ngày, nhưng phát hiện được dấu hiệu session bị chiếm quyền (session hijacking) để yêu cầu xác thực bổ sung đúng lúc mà không làm phiền người dùng hợp lệ.

**Yêu cầu cụ thể:**
- Session "remember me" (giữ đăng nhập lâu dài) phải được phân tách với các hành động nhạy cảm (thanh toán, đổi địa chỉ giao hàng, đổi phương thức thanh toán) — những hành động này yêu cầu xác thực bổ sung dù session chính vẫn còn hạn.
- Hệ thống phải theo dõi tín hiệu bất thường trong phiên (thay đổi IP/vị trí địa lý giữa các request liên tiếp một cách bất hợp lý, thay đổi user-agent giữa chừng) để gắn cờ session nghi ngờ bị đánh cắp.
- Khi một session bị gắn cờ nghi ngờ, hệ thống phải yêu cầu xác thực lại trước khi cho tiếp tục các hành động nhạy cảm, đồng thời không tự động đăng xuất ngay lập tức nếu chỉ đang xem sản phẩm (tránh làm phiền người dùng hợp lệ vì false positive).
- Khi phát hiện thanh toán được thực hiện ngay sau một thay đổi thông tin nhạy cảm gần đây (ví dụ vừa đổi địa chỉ giao hàng/phương thức thanh toán), phải tự động yêu cầu xác thực lại trước khi cho hoàn tất giao dịch, để chặn kịch bản chiếm session rồi đổi địa chỉ nhận hàng.
- Cung cấp cho người dùng lịch sử các lần đăng nhập/session gần đây (thiết bị, vị trí ước lượng, thời gian) và cách báo cáo nếu phát hiện session không phải của họ, kèm hành động tự động khóa tạm giao dịch đang treo khi người dùng báo cáo bị chiếm session.

---

# Base ERD — Marketplace với session "remember me" nhưng chưa tách biệt hành động nhạy cảm

Đây là **base**: trạng thái hệ thống marketplace *trước khi* có theo dõi bất thường và step-up cho hành động nhạy cảm. Suy luận từ đề bài, base đã có `USER` với `SESSION` dạng "remember me" (đăng nhập 1 lần, giữ lâu dài), `ADDRESS` và `PAYMENT_METHOD` được lưu sẵn để mua nhanh, và `ORDER` khi thanh toán. Base coi session còn hạn là đủ điều kiện cho mọi hành động — kể cả đổi địa chỉ, đổi phương thức thanh toán, hay thanh toán — không có bước xác thực bổ sung nào, cũng chưa theo dõi tín hiệu bất thường trong phiên.

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    USER ||--o{ ADDRESS : has
    USER ||--o{ PAYMENT_METHOD : has
    USER ||--o{ ORDER : places
    ADDRESS ||--o{ ORDER : "giao tới"
    PAYMENT_METHOD ||--o{ ORDER : "thanh toán bằng"

    USER {
        string id PK
        string email
    }
    SESSION {
        string id PK
        string user_id FK
        boolean remember_me
        string device_name
        string ip_address
        string user_agent
        datetime created_at
        datetime expires_at
    }
    ADDRESS {
        string id PK
        string user_id FK
        string address_line
        boolean is_default
        datetime updated_at
    }
    PAYMENT_METHOD {
        string id PK
        string user_id FK
        string type
        string last4
        boolean is_default
        datetime updated_at
    }
    ORDER {
        string id PK
        string user_id FK
        string address_id FK
        string payment_method_id FK
        decimal amount
        string status "pending|paid|failed"
        datetime created_at
    }
```

---

# Base sequence — Login remember me (session dài hạn, không phân biệt hành động sau này)

Đây là **base**, flow đăng nhập với tuỳ chọn "remember me" ở trạng thái hiện tại: sau khi xác thực, hệ thống cấp 1 session dài hạn (vd 30 ngày), session này được dùng cho mọi hành động sau đó mà không phân biệt mức độ nhạy cảm. Đây là nền để so sánh với yêu cầu 1 của đề bài (cần tách biệt session dài hạn khỏi các hành động nhạy cảm).

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant Auth as Auth Service
    participant DB as Database

    User->>App: Đăng nhập, tick chọn "Ghi nhớ đăng nhập"
    App->>Auth: Xác thực username/password
    Auth-->>App: Xác thực thành công

    App->>DB: INSERT SESSION (remember_me=true, expires_at=now+30 ngày, ip_address, user_agent)
    DB-->>App: Session token dài hạn được tạo

    App-->>User: Đăng nhập thành công, phiên có hiệu lực 30 ngày
    Note over App,DB: Cùng 1 session token này sẽ được dùng cho mọi hành động sau đó, kể cả xem sản phẩm, đổi địa chỉ, đổi phương thức thanh toán, hay thanh toán
```

---

# Base sequence — Checkout (đổi địa chỉ, đổi phương thức thanh toán, thanh toán đều chỉ cần session còn hạn)

Đây là **base**, flow thanh toán ở trạng thái hiện tại: chỉ cần session "remember me" còn hạn là đủ để đổi địa chỉ giao hàng, đổi phương thức thanh toán, và hoàn tất thanh toán, không có bước xác thực bổ sung nào dù đây đều là hành động nhạy cảm. Sơ đồ minh hoạ đúng khoảng trống mà yêu cầu 1 và 4 của đề bài nhắm tới: nếu 1 session bị chiếm quyền, kẻ tấn công có thể đổi địa chỉ nhận hàng rồi thanh toán ngay mà không gặp trở ngại nào.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant DB as Database

    User->>App: Mở app với session remember-me còn hạn
    App->>DB: Kiểm tra SESSION còn hạn (expires_at > now)
    DB-->>App: Session hợp lệ

    User->>App: Đổi địa chỉ giao hàng sang địa chỉ mới
    App->>DB: UPDATE ADDRESS, chỉ cần session còn hạn, không hỏi lại gì thêm
    DB-->>App: Đã cập nhật địa chỉ

    User->>App: Đổi phương thức thanh toán sang thẻ mới
    App->>DB: UPDATE PAYMENT_METHOD, chỉ cần session còn hạn
    DB-->>App: Đã cập nhật phương thức thanh toán

    User->>App: Nhấn thanh toán đơn hàng
    App->>DB: INSERT ORDER (status=paid), chỉ cần session còn hạn
    DB-->>App: Thanh toán thành công ngay lập tức
    App-->>User: Đơn hàng hoàn tất, giao tới địa chỉ vừa đổi, trả bằng thẻ vừa đổi
```

---

# Enhance ERD — Tách session dài hạn khỏi hành động nhạy cảm, phát hiện bất thường, step-up và báo cáo session

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `SESSION` thêm các trường theo dõi rủi ro, và có 3 entity mới, ứng trực tiếp với các yêu cầu:

- `SENSITIVE_ACTION_VERIFICATION` (mới) — mọi hành động nhạy cảm (`checkout_payment`, `change_address`, `change_payment_method`) cần 1 bản ghi xác minh riêng dù `SESSION` còn hạn — đáp ứng yêu cầu 1.
- `SESSION_ACTIVITY_LOG` (mới) cùng `SESSION.status`/`flagged_reason` — ghi lại IP/geo/user-agent theo từng request để phát hiện thay đổi bất hợp lý giữa các request liên tiếp, gắn cờ session nghi ngờ nhưng không đăng xuất ngay nếu chỉ đang xem sản phẩm — đáp ứng yêu cầu 2 và 3.
- `SENSITIVE_ACTION_VERIFICATION.trigger_reason=payment_after_recent_change` cùng `ADDRESS.updated_at`/`PAYMENT_METHOD.updated_at` — phát hiện thanh toán diễn ra ngay sau khi vừa đổi thông tin nhạy cảm, buộc xác minh lại trước khi hoàn tất — đáp ứng yêu cầu 4.
- `SESSION_REPORT` (mới) cùng `ORDER.locked_for_review` — người dùng xem lịch sử session và báo cáo session lạ, hệ thống tự động khoá tạm các giao dịch đang treo của session đó — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    USER ||--o{ ADDRESS : has
    USER ||--o{ PAYMENT_METHOD : has
    USER ||--o{ ORDER : places
    ADDRESS ||--o{ ORDER : "giao tới"
    PAYMENT_METHOD ||--o{ ORDER : "thanh toán bằng"
    SESSION ||--o{ SESSION_ACTIVITY_LOG : "ghi mọi request"
    SESSION ||--o{ SENSITIVE_ACTION_VERIFICATION : "yêu cầu xác minh khi"
    SESSION ||--o| SESSION_REPORT : "có thể bị báo cáo"
    ORDER ||--o| SENSITIVE_ACTION_VERIFICATION : "gắn với lần thanh toán"

    USER {
        string id PK
        string email
    }
    SESSION {
        string id PK
        string user_id FK
        boolean remember_me
        string device_name
        string ip_address
        string estimated_city
        string estimated_country
        string user_agent
        string status "active | flagged_suspicious | revoked"
        string flagged_reason "ip_geo_jump | user_agent_changed | reported_by_user"
        datetime created_at
        datetime last_seen_at
        datetime expires_at
    }
    SESSION_ACTIVITY_LOG {
        string id PK
        string session_id FK
        string ip_address
        string estimated_city
        string user_agent
        datetime requested_at
    }
    SENSITIVE_ACTION_VERIFICATION {
        string id PK
        string session_id FK
        string order_id FK
        string action_type "checkout_payment | change_address | change_payment_method"
        string trigger_reason "sensitive_action | session_flagged_suspicious | payment_after_recent_change"
        string status "pending | verified | failed"
        datetime requested_at
        datetime verified_at
    }
    SESSION_REPORT {
        string id PK
        string session_id FK
        string reported_by_user_id FK
        datetime reported_at
        string action_taken "session_revoked_and_orders_locked"
    }
    ADDRESS {
        string id PK
        string user_id FK
        string address_line
        boolean is_default
        datetime updated_at
    }
    PAYMENT_METHOD {
        string id PK
        string user_id FK
        string type
        string last4
        boolean is_default
        datetime updated_at
    }
    ORDER {
        string id PK
        string user_id FK
        string address_id FK
        string payment_method_id FK
        decimal amount
        string status "pending|paid|failed"
        boolean locked_for_review
        string locked_reason
        datetime created_at
    }
```

---

# Enhance sequence — Checkout (hành động nhạy cảm luôn cần xác minh bổ sung dù session còn hạn)

Đây là **enhance** của flow `checkout` đã có ở base. So với base (chỉ cần session còn hạn là đổi địa chỉ/phương thức thanh toán/thanh toán được ngay), nay mỗi hành động nhạy cảm đều tạo 1 `SENSITIVE_ACTION_VERIFICATION` riêng và bắt buộc xác minh lại (vd mã OTP, xác nhận sinh trắc học) trước khi được thực hiện, bất kể session chính vẫn còn hạn. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant Verify as Verification Service
    participant DB as Database

    User->>App: Mở app với session remember-me còn hạn
    App->>DB: Kiểm tra SESSION còn hạn và status=active
    DB-->>App: Session hợp lệ, cho phép duyệt sản phẩm bình thường

    User->>App: Đổi địa chỉ giao hàng sang địa chỉ mới
    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=change_address, trigger_reason=sensitive_action)
    Verify->>User: Yêu cầu xác minh lại (OTP gửi qua kênh đã đăng ký)
    User-->>Verify: Nhập đúng OTP
    Verify->>DB: UPDATE SENSITIVE_ACTION_VERIFICATION status=verified
    Verify->>DB: UPDATE ADDRESS (updated_at=now())
    DB-->>App: Địa chỉ đã được cập nhật

    User->>App: Đổi phương thức thanh toán sang thẻ mới
    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=change_payment_method, trigger_reason=sensitive_action)
    Verify->>User: Yêu cầu xác minh lại lần nữa
    User-->>Verify: Nhập đúng OTP
    Verify->>DB: UPDATE SENSITIVE_ACTION_VERIFICATION status=verified
    Verify->>DB: UPDATE PAYMENT_METHOD (updated_at=now())
    DB-->>App: Phương thức thanh toán đã được cập nhật

    User->>App: Nhấn thanh toán đơn hàng
    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=checkout_payment, trigger_reason=sensitive_action)
    Verify->>User: Yêu cầu xác minh lại trước khi hoàn tất thanh toán
    User-->>Verify: Xác minh thành công
    Verify->>DB: INSERT ORDER (status=paid)
    DB-->>App: Thanh toán thành công
    App-->>User: Đơn hàng hoàn tất

    Note over App,Verify: Dù session dài hạn còn hạn tới 30 ngày, mỗi hành động nhạy cảm vẫn đòi hỏi 1 lần xác minh riêng, tách biệt hoàn toàn khỏi trạng thái "đã đăng nhập"
```

---

# Enhance sequence — Phát hiện bất thường trong phiên, gắn cờ nghi ngờ mà không đăng xuất ngay

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Mỗi request đều được ghi vào `SESSION_ACTIVITY_LOG` để so sánh IP/vị trí địa lý và user-agent giữa các request liên tiếp, phát hiện dấu hiệu session bị chiếm quyền. Khi bị gắn cờ nghi ngờ, hệ thống chỉ chặn hành động nhạy cảm chứ không đăng xuất người dùng nếu họ chỉ đang xem sản phẩm. Đáp ứng yêu cầu 2 và 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant Monitor as Session Anomaly Monitor
    participant DB as Database

    User->>App: Request 1 - Xem sản phẩm (IP tại Hà Nội, user-agent Chrome/Android)
    App->>DB: Ghi SESSION_ACTIVITY_LOG (ip, estimated_city=Hà Nội, user_agent=Chrome/Android, requested_at)

    Note over User,App: 5 phút sau, cùng session token nhưng request đến từ nơi khác

    User->>App: Request 2 - Xem sản phẩm khác (IP tại một quốc gia khác, user-agent Firefox/Windows)
    App->>DB: Ghi SESSION_ACTIVITY_LOG (ip mới, estimated_city khác, user_agent khác, requested_at)

    App->>Monitor: So sánh activity log request liên tiếp gần nhất của session
    Monitor->>Monitor: Phát hiện đổi vị trí địa lý bất hợp lý trong 5 phút (không thể di chuyển vật lý), đồng thời đổi user-agent giữa chừng

    Monitor->>DB: UPDATE SESSION SET status=flagged_suspicious, flagged_reason=ip_geo_jump

    alt Request tiếp theo chỉ là xem sản phẩm/duyệt danh mục
        App-->>User: Vẫn cho xem bình thường, không đăng xuất, không làm phiền
        Note over App,Monitor: Tránh false positive làm phiền người dùng hợp lệ (vd đổi mạng, dùng VPN) khi họ chưa chạm hành động nhạy cảm nào
    else Request tiếp theo là hành động nhạy cảm (đổi địa chỉ, đổi thanh toán, checkout)
        App->>App: Chặn lại, chuyển sang flow xác minh bổ sung bắt buộc trước khi cho tiếp tục
        App-->>User: "Phát hiện dấu hiệu bất thường, vui lòng xác minh lại trước khi tiếp tục"
    end
```

---

# Enhance sequence — Tự động yêu cầu xác minh lại khi thanh toán ngay sau khi vừa đổi thông tin nhạy cảm

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Ngay cả khi session không bị gắn cờ nghi ngờ và người dùng đã xác minh xong cho lần đổi địa chỉ/phương thức thanh toán trước đó, hệ thống vẫn tự động buộc xác minh lại thêm 1 lần nữa nếu thanh toán diễn ra ngay sau đó, để chặn kịch bản chiếm session rồi đổi địa chỉ nhận hàng rồi thanh toán liền. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ chiếm session (hoặc User hợp lệ)
    participant App as Marketplace App
    participant Verify as Verification Service
    participant DB as Database

    Attacker->>App: Đổi địa chỉ giao hàng sang địa chỉ lạ
    App->>Verify: Yêu cầu xác minh cho change_address, xác minh thành công
    Verify->>DB: UPDATE ADDRESS (updated_at=now())

    Note over Attacker,App: Chỉ 30 giây sau, cùng session này thực hiện thanh toán

    Attacker->>App: Nhấn thanh toán đơn hàng giá trị lớn
    App->>DB: Kiểm tra ADDRESS.updated_at và PAYMENT_METHOD.updated_at của đơn hàng này
    DB-->>App: address vừa được đổi cách đây 30 giây, nằm trong ngưỡng "vừa thay đổi gần đây" (vd dưới 15 phút)

    App->>Verify: Tạo SENSITIVE_ACTION_VERIFICATION(action_type=checkout_payment, trigger_reason=payment_after_recent_change)
    Note over App,Verify: Bắt buộc xác minh lại từ đầu, KHÔNG tính là đã xác minh trong cùng phiên dù lần đổi địa chỉ trước đó vừa xác minh xong

    Verify->>Attacker: Yêu cầu xác minh bổ sung trước khi hoàn tất thanh toán
    alt Xác minh thất bại (không đúng OTP, không phải sinh trắc học của chủ tài khoản)
        Verify-->>App: Từ chối, không cho hoàn tất thanh toán
        App-->>Attacker: Giao dịch bị chặn, tài khoản thật không mất tiền
    else Xác minh thành công (đúng là chủ tài khoản)
        Verify->>DB: INSERT ORDER (status=paid)
        DB-->>App: Thanh toán thành công
        App-->>Attacker: Đơn hàng hoàn tất
    end
```

---

# Enhance sequence — Xem lịch sử session, báo cáo session lạ, tự động khoá giao dịch treo

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Người dùng xem được danh sách session gần đây (thiết bị, vị trí ước lượng, thời gian) và có thể báo cáo session không phải của mình, hệ thống sẽ tự động thu hồi session đó và khoá tạm các giao dịch đang treo liên quan. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant DB as Database
    participant Report as Session Report Handler

    User->>App: Mở màn hình "Thiết bị và phiên đăng nhập"
    App->>DB: SELECT SESSION WHERE user_id=... ORDER BY last_seen_at DESC
    DB-->>App: Danh sách session (device_name, estimated_city, estimated_country, created_at, last_seen_at, status)
    App-->>User: Hiển thị danh sách kèm nút "Đây không phải tôi" cho từng session

    User->>App: Chọn báo cáo 1 session lạ (vd session tại thành phố người dùng chưa từng tới)
    App->>Report: Tạo SESSION_REPORT (session_id, reported_by_user_id, reported_at)

    Report->>DB: UPDATE SESSION SET status=revoked, flagged_reason=reported_by_user
    Report->>DB: SELECT ORDER WHERE session liên quan AND status=pending

    alt Có giao dịch đang treo được tạo bởi session bị báo cáo
        Report->>DB: UPDATE ORDER SET locked_for_review=true, locked_reason=session_reported_by_owner
        Report-->>User: "Đã khoá tạm N giao dịch đang treo liên quan, đội hỗ trợ sẽ liên hệ xác minh"
    else Không có giao dịch nào đang treo
        Report-->>User: "Đã thu hồi session, không có giao dịch nào bị ảnh hưởng"
    end

    Report->>DB: Ghi SESSION_REPORT.action_taken=session_revoked_and_orders_locked
    App-->>User: Session bị báo cáo không còn dùng được nữa ngay lập tức
```
