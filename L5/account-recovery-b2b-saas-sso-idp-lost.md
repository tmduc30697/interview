# SaaS B2B khôi phục quyền truy cập khi công ty khách hàng đổi hoặc mất Identity Provider

**Hệ thống:** Một SaaS doanh nghiệp mà phần lớn tài khoản đăng nhập qua SSO của công ty khách hàng (Okta/Azure AD...), không có mật khẩu riêng trên nền tảng.

**Vai trò của flow:** Xử lý tình huống công ty khách hàng đổi nhà cung cấp IdP, mất quyền quản trị IdP cũ, hoặc IdP cũ ngừng hoạt động — khiến toàn bộ nhân viên công ty đó không thể đăng nhập qua SSO như bình thường, cần một quy trình khôi phục quyền truy cập không phụ thuộc vào chính IdP đã hỏng.

**Yêu cầu cụ thể:**
- Vì tài khoản SSO-only không có mật khẩu, khi IdP của cả một tổ chức khách hàng ngừng hoạt động, toàn bộ nhân viên công ty đó bị khóa cùng lúc — cần một cơ chế khôi phục cấp tổ chức, không phải xử lý từng user riêng lẻ, để tránh hàng trăm ticket hỗ trợ cùng lúc và tránh gián đoạn nghiệp vụ kéo dài.
- Việc xác minh ai có quyền yêu cầu chuyển đổi hoặc cấu hình lại SSO cho một tổ chức phải chặt chẽ, để tránh kẻ tấn công giả danh admin công ty khách hàng nhằm chiếm quyền truy cập toàn bộ dữ liệu tổ chức đó thông qua việc trỏ SSO sang một IdP do chúng kiểm soát.
- Cần thiết lập sẵn từ trước một tài khoản break-glass cấp tổ chức cho ít nhất một admin của mỗi tổ chức khách hàng lớn, không phải tạo mới lúc khẩn cấp, để có đường vào dự phòng khi SSO chính hỏng hoàn toàn, kèm quy trình giám sát chặt việc sử dụng tài khoản này.
- Khi công ty khách hàng chủ động đổi IdP theo kế hoạch (không phải sự cố), quy trình cutover phải cho phép chạy song song cả IdP cũ và mới trong một khoảng thời gian chuyển tiếp, để tránh có khoảnh khắc không IdP nào hoạt động khiến toàn bộ nhân viên bị khóa giữa chừng.
- Phải phân biệt rõ giữa lỗi tạm thời của IdP (downtime ngắn, sự cố mạng) và việc tổ chức thực sự cần chuyển IdP, tránh kích hoạt quy trình khôi phục khẩn cấp cấp tổ chức — vốn có rủi ro bảo mật cao hơn — chỉ vì một sự cố ngắn hạn có thể tự phục hồi.

---

# Base ERD — SSO-only B2B SaaS trước khi có cơ chế khôi phục

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có yêu cầu khôi phục quyền truy cập cấp tổ chức. Đề bài nói tài khoản "SSO-only, không có mật khẩu riêng" và mỗi công ty khách hàng dùng 1 IdP để đăng nhập — nên base chỉ cần đủ 4 entity nền tảng để việc "IdP hỏng → cả tổ chức bị khóa" có ý nghĩa: tổ chức, cấu hình IdP (đúng 1 cấu hình đang active mỗi tổ chức), user thuộc tổ chức, và session đăng nhập. Chưa có bất kỳ entity nào phục vụ khôi phục/break-glass/cutover — những thứ đó là phần enhance.

```mermaid
erDiagram
    ORGANIZATION ||--o{ USER : employs
    ORGANIZATION ||--|| IDP_CONFIG : "configured with (1 active)"
    USER ||--o{ SESSION : creates

    ORGANIZATION {
        string id PK
        string name
        string status
    }
    IDP_CONFIG {
        string id PK
        string org_id FK
        string provider_type
        string metadata_url
        string signing_cert
        string status "active"
    }
    USER {
        string id PK
        string org_id FK
        string email
        string role
        string external_idp_subject_id
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
```

---

# Base sequence — SSO Login

Đây là **base**, flow "Đăng nhập qua SSO" — tiền đề trực tiếp cho enhance: chính vì mọi nhân viên chỉ đăng nhập được qua **đúng 1** IdP đang active của tổ chức, nên khi IdP đó hỏng, toàn bộ tổ chức bị khóa cùng lúc (vấn đề gốc mà đề bài yêu cầu giải quyết).

```mermaid
sequenceDiagram
    actor Employee
    participant App as SaaS App (SP)
    participant DB as IDP_CONFIG store
    participant IdP as Org's IdP

    Employee->>App: Truy cập app bằng email công ty
    App->>DB: Tìm IDP_CONFIG active theo org (suy ra từ domain email)
    DB-->>App: Trả về 1 IDP_CONFIG active
    App-->>Employee: Redirect sang IdP để xác thực
    Employee->>IdP: Đăng nhập (username/password/MFA tại IdP)
    IdP-->>App: Trả assertion (SAML/OIDC) đã ký
    App->>App: Verify chữ ký assertion bằng cert trong IDP_CONFIG
    App->>App: Tìm/khớp USER theo external_idp_subject_id
    App->>App: Tạo SESSION mới
    App-->>Employee: Đăng nhập thành công
```

---

# Base sequence — Configure IdP

Đây là **base**, flow "Admin cấu hình lại IdP cho tổ chức" ở trạng thái hiện tại — chưa có xác minh chặt chẽ. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài ("việc xác minh ai có quyền yêu cầu chuyển đổi/cấu hình lại SSO phải chặt chẽ") chính là siết lại flow này để tránh kẻ tấn công giả danh admin trỏ SSO sang IdP do chúng kiểm soát.

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Console as SaaS Admin Console
    participant DB as IDP_CONFIG store

    Admin->>Console: Đăng nhập (qua SSO hiện tại)
    Admin->>Console: Mở màn hình SSO Settings
    Admin->>Console: Nhập metadata IdP mới (entity ID, cert, endpoint)
    Console->>Console: Validate định dạng metadata
    Console->>DB: Ghi đè IDP_CONFIG đang active của org
    DB-->>Console: Cập nhật thành công
    Console-->>Admin: Xác nhận cấu hình mới có hiệu lực ngay
    Note over DB: Không có bước xác minh danh tính/approval bổ sung
```

---

# Enhance ERD — sau khi có cơ chế khôi phục cấp tổ chức

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `IDP_CONFIG` chuyển từ 1-1 (1 active/org) sang 1-N, thêm `status` (active/retiring/retired/candidate) và khoảng hiệu lực — để hỗ trợ chạy song song 2 IdP trong lúc cutover có kế hoạch.
- `BREAK_GLASS_ACCOUNT` (mới) — tài khoản cấp tổ chức được thiết lập sẵn từ trước, không tạo lúc khẩn cấp, có MFA riêng và theo dõi lần dùng gần nhất.
- `SSO_CHANGE_REQUEST` (mới) — mọi yêu cầu đổi/cấu hình lại IdP (dù chủ động hay khẩn cấp) đều phải đi qua 1 request có xác minh + trạng thái duyệt, thay vì ghi đè trực tiếp như base.
- `OUTAGE_INCIDENT` (mới) — ghi nhận và phân loại sự cố IdP (tạm thời hay thực sự cần chuyển) trước khi cho phép mở `SSO_CHANGE_REQUEST` loại khẩn cấp.
- `AUDIT_LOG` (mới) — giám sát chặt việc sử dụng break-glass account.

```mermaid
erDiagram
    ORGANIZATION ||--o{ USER : employs
    ORGANIZATION ||--o{ IDP_CONFIG : "configured with (nhiều, theo status)"
    ORGANIZATION ||--o{ BREAK_GLASS_ACCOUNT : "provisioned for"
    ORGANIZATION ||--o{ SSO_CHANGE_REQUEST : requests
    ORGANIZATION ||--o{ OUTAGE_INCIDENT : experiences
    USER ||--o{ SESSION : creates
    BREAK_GLASS_ACCOUNT ||--o{ AUDIT_LOG : "usage logged in"
    SSO_CHANGE_REQUEST ||--o| OUTAGE_INCIDENT : "may cite"
    SSO_CHANGE_REQUEST ||--o| IDP_CONFIG : "produces/updates"

    ORGANIZATION {
        string id PK
        string name
        string status
    }
    IDP_CONFIG {
        string id PK
        string org_id FK
        string provider_type
        string metadata_url
        string signing_cert
        string status "active | retiring | retired | candidate"
        datetime valid_from
        datetime valid_until
    }
    USER {
        string id PK
        string org_id FK
        string email
        string role
        string external_idp_subject_id
    }
    SESSION {
        string id PK
        string user_id FK
        datetime created_at
        datetime expires_at
    }
    BREAK_GLASS_ACCOUNT {
        string id PK
        string org_id FK
        string credential_hash
        string mfa_secret
        string status
        datetime provisioned_at
        datetime last_used_at
    }
    SSO_CHANGE_REQUEST {
        string id PK
        string org_id FK
        string type "planned_cutover | emergency_recovery"
        string requested_by
        string verification_status
        string approval_status
        datetime created_at
        datetime cutover_start
        datetime cutover_end
    }
    OUTAGE_INCIDENT {
        string id PK
        string org_id FK
        string idp_config_id FK
        datetime detected_at
        int duration_minutes
        string classification "transient | confirmed_outage"
        datetime resolved_at
    }
    AUDIT_LOG {
        string id PK
        string break_glass_account_id FK
        string action
        datetime created_at
    }
```

---

# Enhance sequence — SSO Login

Đây là **enhance**, cùng flow "SSO Login" đã có ở base nhưng nay thay đổi để hỗ trợ yêu cầu "chạy song song IdP cũ và mới trong thời gian chuyển tiếp": app không còn giả định org chỉ có đúng 1 `IDP_CONFIG` active, mà phải khớp assertion trả về với đúng cấu hình (cũ hoặc mới) đang hiệu lực tại thời điểm đó — nhờ vậy không có khoảnh khắc nào cả tổ chức bị khóa giữa chừng khi cutover có kế hoạch.

```mermaid
sequenceDiagram
    actor Employee
    participant App as SaaS App (SP)
    participant DB as IDP_CONFIG store
    participant IdP as Old hoặc New IdP

    Employee->>App: Truy cập app bằng email công ty
    App->>DB: Tìm toàn bộ IDP_CONFIG có status active/retiring của org
    DB-->>App: Trả về 1 hoặc 2 config (song song trong lúc cutover)
    App-->>Employee: Redirect sang IdP tương ứng (theo lựa chọn/home realm discovery)
    Employee->>IdP: Đăng nhập tại IdP (cũ hoặc mới)
    IdP-->>App: Trả assertion đã ký, kèm issuer
    App->>DB: Khớp issuer với đúng IDP_CONFIG (cũ hoặc mới) đang hiệu lực
    App->>App: Verify chữ ký assertion bằng cert của config vừa khớp
    App->>App: Tìm/khớp USER, tạo SESSION
    App-->>Employee: Đăng nhập thành công dù đang trong giai đoạn cutover
```

---

# Enhance sequence — Configure IdP

Đây là **enhance**, cùng flow "Admin cấu hình lại IdP" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 2 của đề bài: không còn ghi đè trực tiếp `IDP_CONFIG`, mà phải qua `SSO_CHANGE_REQUEST` với xác minh danh tính chặt chẽ + duyệt nội bộ, để tránh kẻ tấn công giả danh admin trỏ SSO sang IdP do chúng kiểm soát.

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Console as SaaS Admin Console
    participant Verify as Identity Verification Service
    participant Security as Internal Security Review
    participant DB as SSO_CHANGE_REQUEST / IDP_CONFIG store

    Admin->>Console: Yêu cầu đổi/cấu hình lại IdP, nộp metadata IdP mới
    Console->>Verify: Xác minh danh tính admin (out-of-band: xác nhận qua contact đã đăng ký của tổ chức, không chỉ session hiện tại)
    Verify-->>Console: Kết quả xác minh
    alt Xác minh thất bại hoặc đáng ngờ
        Console-->>Admin: Từ chối, yêu cầu xác minh lại/qua kênh khác
    else Xác minh hợp lệ
        Console->>DB: Tạo SSO_CHANGE_REQUEST (type=planned_cutover, verification_status=verified)
        DB->>Security: Đưa vào hàng chờ duyệt nội bộ
        Security-->>DB: Duyệt request
        DB->>DB: Tạo IDP_CONFIG mới với status=candidate
        DB-->>Console: Request approved, sẵn sàng cho bước cutover
        Console-->>Admin: Thông báo request đã duyệt
    end
```

---

# Enhance sequence — Planned IdP cutover

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 4 của đề bài: khi tổ chức chủ động đổi IdP theo kế hoạch, cho phép chạy song song IdP cũ và mới trong 1 khoảng thời gian chuyển tiếp, để không có khoảnh khắc nào cả tổ chức bị khóa. Flow này tiếp nối ngay sau khi `SSO_CHANGE_REQUEST` được duyệt ở flow "Configure IdP".

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Console as SaaS Admin Console
    participant DB as IDP_CONFIG store
    participant Monitor as Migration Monitor
    actor Employees

    Console->>DB: Kích hoạt IDP_CONFIG mới (status=active), IDP_CONFIG cũ chuyển status=retiring
    Note over DB: Cả 2 config cùng hiệu lực trong cutover window (valid_from/valid_until)
    Employees->>DB: Đăng nhập qua IdP cũ hoặc mới (xem flow SSO Login sau enhance)
    Monitor->>DB: Theo dõi tỉ lệ đăng nhập thành công theo từng IDP_CONFIG
    Monitor-->>Admin: Báo cáo tiến độ migrate nhân viên sang IdP mới
    Admin->>Console: Xác nhận đủ điều kiện kết thúc cutover (đa số đã chuyển, hoặc hết hạn window)
    Console->>DB: Đặt IDP_CONFIG cũ status=retired
    Console->>DB: Cập nhật SSO_CHANGE_REQUEST status=completed
    DB-->>Admin: Cutover hoàn tất, chỉ còn IdP mới active
```

---

# Enhance sequence — Emergency org-wide recovery (break-glass)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đây là flow trung tâm của đề bài, gộp 3 yêu cầu còn lại: (1) khôi phục cấp tổ chức thay vì xử lý từng user khi IdP hỏng hoàn toàn, (2) dùng `BREAK_GLASS_ACCOUNT` đã thiết lập sẵn từ trước kèm giám sát chặt (`AUDIT_LOG`), và (3) phải phân biệt sự cố tạm thời với việc thực sự cần khôi phục khẩn cấp trước khi kích hoạt quy trình rủi ro cao này.

```mermaid
sequenceDiagram
    actor Employees
    participant App as SaaS App
    participant Monitor as Outage Monitor
    participant Incident as OUTAGE_INCIDENT store
    actor BGAdmin as Org Admin (dùng break-glass)
    participant BG as BREAK_GLASS_ACCOUNT store
    participant Audit as AUDIT_LOG
    participant DB as SSO_CHANGE_REQUEST / IDP_CONFIG store

    Employees->>App: Thử đăng nhập qua IdP của org
    App-->>Monitor: Báo lỗi xác thực/timeout hàng loạt từ cùng 1 org
    Monitor->>Incident: Tạo/cập nhật OUTAGE_INCIDENT, đo duration
    alt Duration ngắn, tự phục hồi (transient)
        Incident->>Incident: classification=transient
        Monitor-->>BGAdmin: Không kích hoạt quy trình khẩn cấp
    else Duration vượt ngưỡng, không tự phục hồi
        Incident->>Incident: classification=confirmed_outage
        Monitor-->>BGAdmin: Cảnh báo, mở đường vào break-glass cho org
        BGAdmin->>App: Đăng nhập bằng BREAK_GLASS_ACCOUNT + MFA riêng
        App->>BG: Xác thực credential + MFA
        BG-->>App: Hợp lệ
        App->>Audit: Ghi log sử dụng break-glass (actor, thời điểm, hành động)
        BGAdmin->>DB: Tạo SSO_CHANGE_REQUEST (type=emergency_recovery, cite OUTAGE_INCIDENT)
        DB->>DB: Duyệt nhanh (expedited) nhưng vẫn ghi nhận đầy đủ
        DB->>DB: Kích hoạt IDP_CONFIG khôi phục (IdP mới hoặc tạm thời) cho toàn org
        DB-->>Employees: Toàn bộ nhân viên đăng nhập lại được (xem flow SSO Login)
        Note over Audit: Post-incident review dựa trên AUDIT_LOG sau khi khôi phục xong
    end
```
