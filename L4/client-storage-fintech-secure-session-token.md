# Lưu token/session phía client cho ứng dụng ngân hàng số

**Hệ thống:** Ứng dụng ngân hàng số/fintech web lưu access token và dữ liệu phiên đăng nhập ở phía client, phải cân bằng giữa trải nghiệm (không bắt đăng nhập lại liên tục) và yêu cầu bảo mật nghiêm ngặt (không để token bị đánh cắp qua XSS, không rò rỉ khi dùng chung máy).

**Vai trò của flow:** Chọn đúng nơi lưu (memory/sessionStorage/httpOnly cookie) cho từng loại dữ liệu nhạy cảm, và xử lý đúng vòng đời khi đóng tab/đăng xuất/dùng chung máy.

**Yêu cầu cụ thể:**
- Access token ngắn hạn chỉ giữ trong memory (biến JS, không phải localStorage/sessionStorage) để giảm bề mặt tấn công XSS; refresh token dùng cookie httpOnly + Secure + SameSite do server set, JS không đọc trực tiếp được — nêu rõ vì sao localStorage bị loại cho trường hợp này (mọi script chạy trên trang, kể cả script bên thứ ba lỡ bị compromise, đều đọc được).
- Thông tin phi nhạy cảm (tab đang chọn, filter đang áp dụng trên dashboard) có thể lưu sessionStorage — nhưng phải tự xóa sạch mọi state khi logout, kể cả khi logout được trigger từ tab khác (đăng xuất ở 1 tab phải làm mất hiệu lực token ở toàn bộ tab đang mở, không chỉ tab bấm logout).
- Máy dùng chung (quầy giao dịch, máy tính công cộng): sessionStorage tự xóa khi đóng tab/trình duyệt là chưa đủ — cần thêm auto-logout theo thời gian không thao tác (idle timeout) và xóa rõ ràng toàn bộ cache/state khi timeout, không dựa hoàn toàn vào hành vi đóng tab của user.
- Khi vô tình mở 2 tab với 2 tài khoản khác nhau (nhân viên hỗ trợ dùng nhiều tài khoản test) — vì sessionStorage cô lập theo tab nên đây là hành vi đúng, nhưng phải test kỹ để đảm bảo không có state nào vô tình rò rỉ qua cơ chế khác (ví dụ BroadcastChannel dùng chung cho tính năng khác vô tình đồng bộ nhầm session giữa 2 tab khác tài khoản).
- Audit: liệt kê rõ mọi nơi dữ liệu nhạy cảm (số dư, số tài khoản, lịch sử giao dịch) có thể vô tình bị cache lại (browser back/forward cache - bfcache, hoặc cache của service worker nếu có) sau khi logout, và đảm bảo dùng `Cache-Control: no-store` hoặc unload handler để không xem lại được dữ liệu cũ bằng nút Back sau khi đã đăng xuất.

---

# Base ERD — Lưu token phía client trước khi siết bảo mật

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, mô tả trạng thái hệ thống **trước khi** áp các yêu cầu bảo mật nghiêm ngặt. Đề bài ngầm định hệ thống ngân hàng số hiện đã có đăng nhập và lưu token phía client theo cách đơn giản/naive: cả access token lẫn refresh token đều được lưu chung vào localStorage sau khi đăng nhập, không phân loại theo mức độ nhạy cảm, không có idle timeout, không có đồng bộ đăng xuất giữa các tab, chưa xử lý bfcache. Đây chính là những vấn đề mà enhance sẽ giải quyết.

```mermaid
erDiagram
    USER ||--o| CLIENT_LOCAL_SESSION : "stores tokens in localStorage"

    USER {
        string id PK
        string email
    }
    CLIENT_LOCAL_SESSION {
        string id PK
        string user_id FK
        string access_token "lưu trong localStorage, mọi script trên trang đều đọc được"
        string refresh_token "lưu trong localStorage, cùng rủi ro như access_token"
        datetime stored_at
    }
```

---

# Base sequence — Login (cả 2 token đều lưu localStorage)

Đây là **base**, flow đăng nhập ở trạng thái trước enhance. Sau khi xác thực, server trả về cả access token lẫn refresh token, và client lưu cả 2 vào localStorage để dùng cho các lần gọi API sau. Đây chính là điểm mà enhance sẽ thay đổi triệt để: bất kỳ đoạn script nào chạy trên trang, kể cả script bên thứ ba lỡ bị compromise, cũng đọc được các token này.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant API as Auth API
    participant LS as localStorage (CLIENT_LOCAL_SESSION)

    User->>App: Nhập email/mật khẩu, đăng nhập
    App->>API: POST credentials
    API-->>App: Trả về access_token và refresh_token
    App->>LS: Lưu access_token và refresh_token vào localStorage
    Note over LS: Bất kỳ script nào chạy trên trang đều đọc được cả 2 token này
    App-->>User: Chuyển vào dashboard, dùng access_token từ localStorage cho các API call
```

---

# Base sequence — Logout (chỉ xoá tab hiện tại, không có audit cache)

Đây là **base**, flow đăng xuất ở trạng thái trước enhance. Logout chỉ xoá localStorage của đúng tab đang bấm, các tab khác đang mở với cùng tài khoản vẫn còn access_token hợp lệ trong localStorage của chúng (localStorage dùng chung theo origin nhưng flow logout ở base không chủ động thông báo cho tab khác). Ngoài ra không có xử lý gì cho cache-control hay bfcache, nên bấm Back sau khi logout có thể thấy lại trang có dữ liệu nhạy cảm từ bộ nhớ cache trình duyệt.

```mermaid
sequenceDiagram
    actor User as Người dùng (Tab A)
    participant TabA as Tab A
    participant LS as localStorage (dùng chung origin)
    actor OtherUser as Người dùng (Tab B, cùng tài khoản)
    participant TabB as Tab B

    User->>TabA: Xem số dư, lịch sử giao dịch (dùng access_token từ localStorage)
    User->>TabA: Bấm đăng xuất
    TabA->>LS: Xoá access_token, refresh_token khỏi localStorage
    TabA-->>User: Chuyển về màn hình đăng nhập

    Note over TabB: Tab B đang mở cùng tài khoản không nhận được thông báo gì
    OtherUser->>TabB: Vẫn thao tác bình thường ở Tab B với access_token cũ còn hiệu lực

    User->>TabA: Bấm nút Back trên trình duyệt
    TabA-->>User: Có thể hiển thị lại trang số dư/giao dịch từ cache trình duyệt (bfcache), dù đã đăng xuất
```

---

# Enhance ERD — Phân loại nơi lưu theo mức độ nhạy cảm

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base (mọi thứ dồn vào `CLIENT_LOCAL_SESSION` trong localStorage), enhance tách dữ liệu ra nhiều nơi lưu khác nhau theo đúng mức độ nhạy cảm, và thêm cơ chế điều phối vòng đời. Một số entity dưới đây **không phải bảng DB thật** mà là cấu trúc sống trong bộ nhớ trình duyệt/cookie — được mô hình hoá dạng ERD để thấy rõ dữ liệu nào nằm ở đâu:

- `ACCESS_TOKEN_MEMORY` (mới, chỉ tồn tại trong biến JS ở RAM) — không phải localStorage/sessionStorage, mất khi reload trang, giảm bề mặt tấn công XSS vì script không tồn tại nào đọc được sau khi trang tải lại — yêu cầu 1.
- `REFRESH_TOKEN_COOKIE` (mới, cookie httpOnly + Secure + SameSite do server set) — JS không đọc trực tiếp được, khác hẳn base lưu refresh_token vào localStorage mà mọi script kể cả script bên thứ ba compromised đều đọc được — yêu cầu 1.
- `SERVER_SESSION` (mới, bảng thật trên server) — cho phép server thu hồi refresh token tập trung, làm nền cho việc đăng xuất có hiệu lực ở mọi tab/thiết bị — yêu cầu 2.
- `SESSION_UI_STATE` (mới, sessionStorage, cô lập theo tab) — dữ liệu phi nhạy cảm như tab đang chọn, filter dashboard — yêu cầu 2, 4.
- `AUTH_BROADCAST_EVENT` (mới, kênh BroadcastChannel riêng cho auth, đặt tên tách biệt khỏi kênh của tính năng khác) — phát sự kiện logout tới mọi tab cùng origin — yêu cầu 2, 4.
- `IDLE_TIMEOUT_STATE` (mới, bộ nhớ JS theo tab) — theo dõi thời gian không thao tác để tự động đăng xuất trên máy dùng chung — yêu cầu 3.
- `CACHE_AUDIT_ENTRY` (mới, bảng ghi nhận nội bộ dùng cho audit bảo mật) — liệt kê nơi dữ liệu nhạy cảm có thể vô tình bị cache lại (bfcache, service worker cache) và biện pháp chặn — yêu cầu 5.

```mermaid
erDiagram
    USER ||--o{ SERVER_SESSION : "has sessions"
    SERVER_SESSION ||--o| REFRESH_TOKEN_COOKIE : "backs (httpOnly cookie)"
    USER ||--o| ACCESS_TOKEN_MEMORY : "holds in-memory (per tab)"
    USER ||--o| SESSION_UI_STATE : "holds in sessionStorage (per tab)"
    USER ||--o{ AUTH_BROADCAST_EVENT : "emits on logout"
    USER ||--o| IDLE_TIMEOUT_STATE : "tracked per tab"
    USER ||--o{ CACHE_AUDIT_ENTRY : "sensitive surfaces reviewed for"

    USER {
        string id PK
        string email
    }
    SERVER_SESSION {
        string id PK
        string user_id FK
        string refresh_token_hash
        string device_info
        datetime issued_at
        datetime revoked_at
    }
    REFRESH_TOKEN_COOKIE {
        string session_id FK
        string cookie_attributes "httpOnly, Secure, SameSite=Strict"
        string note "JS không đọc trực tiếp được, chỉ trình duyệt tự đính kèm khi gọi API refresh"
    }
    ACCESS_TOKEN_MEMORY {
        string tab_id PK
        string user_id FK
        string access_token "chỉ tồn tại trong biến JS, RAM"
        datetime expires_at
        string note "mất khi reload trang, không phải localStorage/sessionStorage"
    }
    SESSION_UI_STATE {
        string tab_id PK
        string user_id FK
        string selected_tab
        string active_filter
    }
    AUTH_BROADCAST_EVENT {
        string id PK
        string origin_tab_id
        string user_id FK
        string event_type "logout | idle_timeout"
        datetime broadcast_at
    }
    IDLE_TIMEOUT_STATE {
        string tab_id PK
        datetime last_activity_at
        int idle_timeout_ms
        boolean warning_shown
    }
    CACHE_AUDIT_ENTRY {
        string id PK
        string surface "bfcache | service_worker_cache | http_cache"
        string sensitive_data_type "balance | account_number | transaction_history"
        string mitigation "Cache-Control no-store | pagehide/unload clear | cache.delete()"
        string status "covered | gap"
    }
```

---

# Enhance sequence — Login (access token chỉ ở memory, refresh token là cookie httpOnly)

Đây là **enhance**, cùng flow "login" như ở base nhưng thay đổi triệt để nơi lưu 2 loại token. Access token ngắn hạn chỉ giữ trong biến JS ở bộ nhớ (không phải localStorage/sessionStorage) để giảm bề mặt tấn công XSS. Refresh token không còn do JS lưu, mà server set thẳng vào cookie httpOnly + Secure + SameSite, JS không đọc trực tiếp được. Đáp ứng yêu cầu 1 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant API as Auth API
    participant Mem as ACCESS_TOKEN_MEMORY (biến JS, RAM)
    participant Cookie as REFRESH_TOKEN_COOKIE (httpOnly, Secure, SameSite)
    participant DB as SERVER_SESSION (server)

    User->>App: Nhập email/mật khẩu, đăng nhập
    App->>API: POST credentials
    API->>DB: Tạo SERVER_SESSION mới, ghi refresh_token_hash
    API-->>App: Trả về access_token trong response body, đồng thời Set-Cookie refresh_token (httpOnly, Secure, SameSite=Strict)
    App->>Mem: Giữ access_token trong biến JS, không ghi vào localStorage/sessionStorage
    Note over Cookie: JS không đọc được nội dung cookie này, trình duyệt tự đính kèm khi gọi API refresh
    Note over Mem: Vì chỉ ở RAM, nếu bị XSS chèn script độc hại thì script cũng không tìm thấy token trong localStorage như base, giảm hẳn bề mặt tấn công so với việc lưu localStorage mà mọi script kể cả script bên thứ ba compromised đều đọc được
    App-->>User: Chuyển vào dashboard
```

---

# Enhance sequence — Logout đồng bộ mọi tab đang mở

Đây là **enhance**, cùng flow "logout" như ở base nhưng thay đổi ở chỗ: đăng xuất ở 1 tab phải làm mất hiệu lực token ở toàn bộ tab đang mở, không chỉ tab bấm logout, và dữ liệu phi nhạy cảm trong sessionStorage cũng phải được xoá sạch ở mọi tab. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng (Tab A, bấm logout)
    participant TabA as Tab A
    participant API as Auth API
    participant DB as SERVER_SESSION
    participant Channel as AUTH_BROADCAST_EVENT (kênh auth riêng)
    participant TabB as Tab B (cùng tài khoản)

    User->>TabA: Bấm đăng xuất
    TabA->>API: POST logout
    API->>DB: Đánh dấu SERVER_SESSION.revoked_at=now, thu hồi refresh token
    API-->>TabA: Xác nhận đã revoke, đồng thời xoá cookie refresh_token
    TabA->>TabA: Xoá ACCESS_TOKEN_MEMORY và SESSION_UI_STATE của chính tab A
    TabA->>Channel: Phát AUTH_BROADCAST_EVENT(event_type=logout, origin_tab_id=A)

    Channel-->>TabB: Nhận sự kiện logout
    TabB->>TabB: Xoá ACCESS_TOKEN_MEMORY và SESSION_UI_STATE của tab B
    TabB-->>User: Chuyển Tab B về màn hình đăng nhập ngay, dù không ai bấm logout ở tab này

    Note over TabB,API: Nếu Tab B cố gọi API bằng access_token cũ trước khi kịp nhận broadcast, server vẫn từ chối vì refresh token cho session đó đã bị revoke
```

---

# Enhance sequence — Auto-logout theo idle timeout cho máy dùng chung

Đây là **enhance**, flow hoàn toàn mới, dành cho máy dùng chung (quầy giao dịch, máy tính công cộng). sessionStorage tự xoá khi đóng tab là chưa đủ vì nhân viên sau có thể dùng lại tab đang mở sẵn; hệ thống cần chủ động đếm thời gian không thao tác và tự đăng xuất, xoá rõ ràng toàn bộ cache/state, không phụ thuộc vào việc user có đóng tab hay không. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant Idle as IDLE_TIMEOUT_STATE
    participant API as Auth API
    participant Mem as ACCESS_TOKEN_MEMORY
    participant Storage as sessionStorage (SESSION_UI_STATE)

    User->>App: Thao tác (click, gõ phím, cuộn trang)
    App->>Idle: Cập nhật last_activity_at=now mỗi khi có thao tác

    loop Kiểm tra định kỳ
        App->>Idle: So sánh now - last_activity_at với idle_timeout_ms
        alt Sắp hết hạn (còn ví dụ 30 giây)
            App-->>User: Hiển thị cảnh báo sắp tự đăng xuất do không thao tác
        else Chưa tới ngưỡng
            App->>App: Không làm gì thêm
        end
    end

    alt Vượt ngưỡng idle_timeout_ms mà không có thao tác nào
        App->>API: Gọi logout tự động (revoke session, tương tự flow cross-tab-logout)
        App->>Mem: Xoá access_token khỏi bộ nhớ
        App->>Storage: Xoá sạch sessionStorage của tab này
        App-->>User: Chuyển về màn hình đăng nhập, ghi rõ lý do tự động đăng xuất do không thao tác
    end
```

---

# Enhance sequence — Kiểm tra cô lập giữa 2 tab dùng 2 tài khoản khác nhau

Đây là **enhance**, flow hoàn toàn mới, mô tả kịch bản kiểm thử khi nhân viên hỗ trợ vô tình mở 2 tab với 2 tài khoản khác nhau để test. Vì sessionStorage và biến bộ nhớ JS cô lập theo từng tab nên đây là hành vi đúng, nhưng phải kiểm tra kỹ để đảm bảo không có state nào rò rỉ qua cơ chế dùng chung khác, ví dụ kênh BroadcastChannel của một tính năng không liên quan tới auth. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor UserX as Nhân viên hỗ trợ (Tab A, tài khoản X)
    participant TabA as Tab A (ACCESS_TOKEN_MEMORY của X, SESSION_UI_STATE của X)
    actor UserY as Nhân viên hỗ trợ (Tab B, tài khoản Y)
    participant TabB as Tab B (ACCESS_TOKEN_MEMORY của Y, SESSION_UI_STATE của Y)
    participant AuthChannel as AUTH_BROADCAST_EVENT (kênh auth, có kiểm tra origin_tab_id/user_id)
    participant OtherChannel as BroadcastChannel của tính năng khác, ví dụ thông báo real-time

    UserX->>TabA: Đăng nhập tài khoản X
    UserY->>TabB: Đăng nhập tài khoản Y ở tab khác

    Note over TabA,TabB: ACCESS_TOKEN_MEMORY và SESSION_UI_STATE cô lập đúng theo từng tab, đây là hành vi mong muốn

    UserX->>TabA: Đăng xuất tài khoản X
    TabA->>AuthChannel: Phát sự kiện logout kèm user_id=X
    AuthChannel-->>TabB: Nhận được sự kiện, nhưng kiểm tra user_id trong event không khớp session hiện tại của Tab B (Y)
    TabB->>TabB: Bỏ qua sự kiện logout của tài khoản X, không tự đăng xuất tài khoản Y

    rect rgb(255, 235, 235)
    Note over OtherChannel: Trường hợp rủi ro cần test kỹ, nếu 1 tính năng khác dùng chung BroadcastChannel không đặt tên riêng và không kiểm tra user_id
    UserX->>TabA: Thao tác khác (ví dụ chọn 1 filter dashboard) vô tình phát qua OtherChannel
    OtherChannel-->>TabB: Nếu kênh này không kiểm tra user_id, Tab B (tài khoản Y) có thể vô tình nhận nhầm state của tài khoản X
    end

    Note over TabA,TabB: Biện pháp, đặt tên kênh BroadcastChannel auth riêng biệt và luôn kèm/kiểm tra user_id trong mọi message trước khi áp dụng ở tab nhận
```

---

# Enhance sequence — Audit cache nhạy cảm và chặn xem lại bằng nút Back sau logout

Đây là **enhance**, cùng ý tưởng flow "logout" ở base nhưng bổ sung phần audit và chặn rò rỉ qua các lớp cache của trình duyệt. Sau khi logout, hệ thống phải đảm bảo không nơi nào (bfcache, service worker cache, HTTP cache) còn giữ lại dữ liệu nhạy cảm như số dư, số tài khoản, lịch sử giao dịch để có thể xem lại bằng nút Back. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant API as Balance/Transaction API
    participant Audit as CACHE_AUDIT_ENTRY
    participant SW as Service Worker cache (nếu có)
    participant Browser as bfcache của trình duyệt

    Note over Audit: Trước khi enhance, đội bảo mật rà soát và liệt kê từng bề mặt có nguy cơ giữ lại dữ liệu nhạy cảm
    Audit->>Audit: Ghi CACHE_AUDIT_ENTRY(surface=bfcache, sensitive_data_type=balance, mitigation=pagehide/unload clear)
    Audit->>Audit: Ghi CACHE_AUDIT_ENTRY(surface=service_worker_cache, sensitive_data_type=transaction_history, mitigation=cache.delete())
    Audit->>Audit: Ghi CACHE_AUDIT_ENTRY(surface=http_cache, sensitive_data_type=account_number, mitigation=Cache-Control no-store)

    User->>App: Xem số dư, lịch sử giao dịch
    App->>API: GET balance, transactions
    API-->>App: Trả dữ liệu kèm header Cache-Control no-store, để trình duyệt không lưu HTTP cache cho các response này

    User->>App: Bấm đăng xuất
    App->>App: Lắng nghe sự kiện pagehide/visibilitychange để chủ động xoá state nhạy cảm khỏi DOM và bộ nhớ trước khi trang có thể bị đưa vào bfcache
    App->>SW: Nếu có service worker cache riêng cho dữ liệu này, chủ động cache.delete() các entry liên quan tài khoản vừa logout

    User->>Browser: Bấm nút Back sau khi đã đăng xuất
    alt Trang được phục hồi từ bfcache
        Browser-->>App: Kích hoạt sự kiện pageshow với persisted=true
        App->>App: Phát hiện không còn access_token hợp lệ trong bộ nhớ, chủ động điều hướng lại về màn hình đăng nhập thay vì hiển thị dữ liệu cũ trong DOM
    else Trang tải lại bình thường
        App->>API: Gọi lại API, bị từ chối vì session đã revoke, điều hướng về đăng nhập
    end
```
