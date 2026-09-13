# Blue-green deployment cho service lõi của nền tảng SaaS multi-tenant

**Hệ thống:** Một nền tảng SaaS B2B phục vụ nhiều tenant (công ty khách hàng) dùng chung hạ tầng, service "core-api" được deploy nhiều lần mỗi tuần.

**Vai trò của flow:** Blue-green deployment cho phép chuyển toàn bộ traffic từ phiên bản cũ (blue) sang phiên bản mới (green) gần như ngay lập tức, và rollback tức thì nếu phát hiện sự cố sau khi chuyển.

**Yêu cầu cụ thể:**
- Trước khi chuyển traffic, phiên bản green phải được deploy đầy đủ song song với blue (không tắt blue), và trải qua bước smoke test tự động (gọi các endpoint quan trọng) xác nhận hoạt động đúng trước khi router bắt đầu chuyển traffic thật sang.
- Việc chuyển traffic từ blue sang green phải là một thao tác gần như tức thời ở tầng router/load balancer (đổi target), không yêu cầu client phải làm gì, và phải có khả năng đảo ngược (chuyển lại về blue) trong vài giây nếu phát hiện lỗi ngay sau khi chuyển.
- Xử lý đúng vấn đề tương thích dữ liệu khi có migration schema DB đi kèm bản deploy mới — phiên bản blue (cũ) vẫn đang chạy phải tiếp tục hoạt động đúng với schema đã thay đổi cho tới khi bị tắt hẳn (yêu cầu migration phải backward-compatible trong giai đoạn chuyển tiếp).
- Định nghĩa rõ tiêu chí tự động rollback (ví dụ tỷ lệ lỗi 5xx vượt ngưỡng X% trong Y phút đầu sau khi chuyển) và cơ chế giám sát phải chạy ngay sau khi chuyển traffic, không chờ phát hiện thủ công từ báo cáo người dùng.
- Đảm bảo các session/kết nối đang mở tại thời điểm chuyển đổi (ví dụ WebSocket, request dài) được xử lý đúng — không bị cắt ngang giữa lúc router đổi target, mà phải hoàn tất trên blue hoặc được chuyển tiếp một cách có kiểm soát.

---

# Base ERD — core-api trước khi có blue-green deployment

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có blue-green deployment. Đề bài nói service "core-api" được deploy nhiều lần/tuần, có migration schema DB đi kèm, và có session/kết nối dài (WebSocket) đang mở — nên base cần đủ: service, deployment (chỉ 1 target đang chạy tại 1 thời điểm), router trỏ thẳng tới target đó, migration DB, và các connection đang mở trên deployment. Chưa có entity nào phục vụ smoke test/switch có thể đảo ngược/rollback policy tự động/connection draining — những thứ đó là phần enhance.

```mermaid
erDiagram
    SERVICE ||--o{ DEPLOYMENT : has
    SERVICE ||--|| ROUTER_CONFIG : "routes via (1 target)"
    SERVICE ||--o{ DB_MIGRATION : applies
    DEPLOYMENT ||--o{ CLIENT_CONNECTION : serves

    SERVICE {
        string id PK
        string name "core-api"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string status "active"
        datetime deployed_at
    }
    ROUTER_CONFIG {
        string id PK
        string service_id FK
        string target_deployment_id FK
    }
    DB_MIGRATION {
        string id PK
        string service_id FK
        string version
        string description
        datetime applied_at
    }
    CLIENT_CONNECTION {
        string id PK
        string deployment_id FK
        string type "websocket | long_request"
        datetime opened_at
        string status
    }
```

---

# Base sequence — Deploy (rolling, chưa có blue-green)

Đây là **base**, flow "Deploy phiên bản mới" ở trạng thái hiện tại — rolling deploy trực tiếp, không có bản song song để test trước. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu 1-3 của đề bài đều nhằm cải tổ lại đúng flow này.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Old as Deployment cũ (đang chạy)
    participant DB as Shared Database
    participant Router as Router/Load Balancer
    actor Client

    CI->>DB: Áp dụng DB_MIGRATION mới trực tiếp lên schema hiện tại
    Note over DB: Không đảm bảo backward-compatible — chỉ cần bản mới chạy đúng
    CI->>Old: Dừng từng instance cũ, khởi động instance phiên bản mới (rolling restart)
    Old-->>Client: Các CLIENT_CONNECTION (WebSocket/request dài) đang mở trên instance bị dừng → bị cắt ngang
    CI->>Router: ROUTER_CONFIG.target_deployment_id trỏ theo instance mới khi rolling xong
    Router-->>Client: Traffic mới đi vào phiên bản mới
    Note over CI,Router: Không có bước smoke test tự động, nếu có lỗi phải chờ dashboard/user report mới phát hiện
```

---

# Base sequence — Rollback (thủ công, chưa có tiêu chí tự động)

Đây là **base**, flow "Xử lý khi bản deploy mới có lỗi" ở trạng thái hiện tại — hoàn toàn thủ công, phát hiện chậm. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 4 của đề bài chính là tự động hoá lại đúng flow này.

```mermaid
sequenceDiagram
    actor Users
    participant Dashboard as Monitoring Dashboard
    actor OnCall as On-call Engineer
    participant CI as CI/CD

    Users->>Users: Gặp lỗi 5xx sau khi bản mới lên
    Users-->>OnCall: Report qua ticket/Slack (không phải phát hiện tự động)
    OnCall->>Dashboard: Vào kiểm tra dashboard, xác nhận thủ công lỗi tăng bất thường
    OnCall->>OnCall: Điều tra, xác định bản deploy mới là nguyên nhân
    OnCall->>CI: Kích hoạt thủ công quy trình rolling deploy để quay lại bản cũ
    CI->>CI: Rolling deploy bản cũ (mất vài phút, giống flow Deploy)
    Note over Users,CI: Trong toàn bộ thời gian này (phát hiện + điều tra + rolling deploy lại), user vẫn gặp lỗi
```

---

# Enhance ERD — sau khi có blue-green deployment

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `DEPLOYMENT` thêm `color` (blue/green) và `status` mở rộng (staging/smoke_testing/live/retiring/rolled_back) — thay vì chỉ có đúng 1 deployment "active".
- `SMOKE_TEST_RUN` (mới) — bắt buộc pass trước khi router chuyển traffic thật.
- `ROUTER_SWITCH_EVENT` (mới) — thao tác chuyển traffic gần như tức thời và có thể đảo ngược (`reversible_until`), thay cho việc router chỉ đơn giản trỏ theo instance đang chạy như base.
- `DB_MIGRATION` thêm `backward_compatible` + `phase` (expand/contract) — đảm bảo blue vẫn chạy đúng trong giai đoạn chuyển tiếp.
- `ROLLBACK_POLICY` + `ROLLBACK_EVENT` (mới) — tiêu chí rollback tự động, không chờ phát hiện thủ công.
- `CONNECTION_DRAIN` (mới) — xử lý có kiểm soát các connection đang mở trên blue khi traffic đã chuyển sang green.

```mermaid
erDiagram
    SERVICE ||--o{ DEPLOYMENT : has
    SERVICE ||--o{ ROUTER_SWITCH_EVENT : records
    SERVICE ||--o{ DB_MIGRATION : applies
    SERVICE ||--|| ROLLBACK_POLICY : "governed by"
    DEPLOYMENT ||--o{ SMOKE_TEST_RUN : "tested by"
    DEPLOYMENT ||--o{ CLIENT_CONNECTION : serves
    DEPLOYMENT ||--o| CONNECTION_DRAIN : "drains via"
    ROUTER_SWITCH_EVENT ||--o| ROLLBACK_EVENT : "may trigger"

    SERVICE {
        string id PK
        string name "core-api"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string color "blue | green"
        string status "staging | smoke_testing | live | retiring | rolled_back"
        datetime deployed_at
    }
    SMOKE_TEST_RUN {
        string id PK
        string deployment_id FK
        string test_name
        string result "pass | fail"
        datetime run_at
    }
    ROUTER_SWITCH_EVENT {
        string id PK
        string service_id FK
        string from_deployment_id FK
        string to_deployment_id FK
        datetime switched_at
        datetime reversible_until
        string status "active | reverted"
    }
    DB_MIGRATION {
        string id PK
        string service_id FK
        string version
        string description
        boolean backward_compatible
        string phase "expand | contract"
        datetime applied_at
    }
    ROLLBACK_POLICY {
        string id PK
        string service_id FK
        decimal error_rate_threshold
        int window_minutes
        string metric_source
    }
    ROLLBACK_EVENT {
        string id PK
        string router_switch_event_id FK
        string triggered_by "auto_policy | manual"
        string reason
        string reverted_to_deployment_id FK
        datetime triggered_at
    }
    CLIENT_CONNECTION {
        string id PK
        string deployment_id FK
        string type "websocket | long_request"
        datetime opened_at
        string status
    }
    CONNECTION_DRAIN {
        string id PK
        string deployment_id FK
        string connection_type
        int count_at_switch
        string drain_status
        datetime completed_at
    }
```

---

# Enhance sequence — Deploy (blue-green, smoke test, migration backward-compatible)

Đây là **enhance**, cùng flow "Deploy" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1-3 của đề bài: green được deploy song song (không tắt blue), migration phải backward-compatible để blue vẫn chạy đúng, smoke test bắt buộc pass trước khi router chuyển traffic thật, và việc chuyển traffic là thao tác tức thời ở router.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Blue as Blue Deployment (đang live)
    participant Green as Green Deployment (mới)
    participant DB as Shared Database
    participant Smoke as Smoke Test Runner
    participant Router as Router/Load Balancer

    CI->>DB: Áp dụng DB_MIGRATION (backward_compatible=true, phase=expand)
    Note over DB: Blue vẫn tiếp tục hoạt động đúng với schema đã đổi trong lúc chuyển tiếp
    CI->>Green: Deploy phiên bản mới thành GREEN, song song với Blue (Blue không bị tắt)
    Green->>Green: status=staging → smoke_testing
    CI->>Smoke: Gọi các endpoint quan trọng lên Green
    Smoke->>DB: Ghi SMOKE_TEST_RUN (test_name, result)
    alt Smoke test fail
        Smoke-->>CI: result=fail
        CI->>Green: Huỷ Green, giữ nguyên Blue đang phục vụ
        CI-->>CI: Deploy bị dừng, không chuyển traffic
    else Smoke test pass
        Smoke-->>CI: result=pass
        Green->>Green: status=live
        CI->>Router: Tạo ROUTER_SWITCH_EVENT (from=Blue, to=Green, reversible_until=+vài giây)
        Router->>Router: Đổi target gần như tức thời — client không cần làm gì
        Router-->>Blue: Ngừng gửi traffic MỚI tới Blue (connection cũ xử lý ở flow "Connection draining")
        Note over Router: Có thể đảo ngược switch trong vài giây tiếp theo (xem flow "Rollback")
    end
```

---

# Enhance sequence — Rollback (tự động, theo tiêu chí lỗi 5xx)

Đây là **enhance**, cùng flow "Rollback" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 4 của đề bài: có `ROLLBACK_POLICY` định nghĩa rõ tiêu chí (tỷ lệ lỗi 5xx vượt ngưỡng X% trong Y phút), và giám sát chạy tự động ngay sau khi router chuyển traffic — không chờ user report như base.

```mermaid
sequenceDiagram
    participant Router as Router/Load Balancer
    participant Monitor as Auto Rollback Monitor
    participant Policy as ROLLBACK_POLICY store
    participant Blue as Blue Deployment (vẫn đang chạy, chưa tắt)
    participant Green as Green Deployment (vừa nhận traffic)

    Note over Router: Ngay sau ROUTER_SWITCH_EVENT chuyển traffic sang Green
    Monitor->>Policy: Lấy error_rate_threshold + window_minutes cho service
    loop Trong suốt window_minutes ngay sau switch
        Monitor->>Green: Theo dõi tỷ lệ lỗi 5xx theo thời gian thực
        alt Tỷ lệ lỗi vượt threshold
            Monitor->>Router: Kích hoạt rollback ngay (không chờ phát hiện thủ công)
            Router->>Router: Tạo ROLLBACK_EVENT (triggered_by=auto_policy), revert target về Blue
            Router-->>Blue: Traffic quay lại Blue (Blue vẫn đang chạy sẵn, không cần khởi động lại)
            Green->>Green: status=rolled_back
            Monitor-->>Monitor: Dừng theo dõi, cảnh báo on-call để điều tra thêm
        else Tỷ lệ lỗi trong ngưỡng cho phép
            Monitor->>Monitor: Tiếp tục theo dõi tới hết window
        end
    end
    Note over Monitor: Hết window mà không vượt threshold → Green được coi là ổn định, Blue chuyển sang retiring
```

---

# Enhance sequence — Connection draining khi chuyển traffic

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (ở base, connection bị cắt ngang thẳng khi instance cũ bị dừng). Đáp ứng yêu cầu thứ 5 của đề bài: session/kết nối đang mở (WebSocket, request dài) tại thời điểm chuyển đổi không bị cắt ngang, mà hoàn tất trên Blue hoặc được chuyển tiếp có kiểm soát.

```mermaid
sequenceDiagram
    participant Router as Router/Load Balancer
    participant Blue as Blue Deployment
    participant Green as Green Deployment
    actor Client

    Note over Router: Ngay sau ROUTER_SWITCH_EVENT (traffic mới đã trỏ sang Green)
    Router->>Blue: Ngừng route request MỚI tới Blue, nhưng KHÔNG kill Blue ngay
    Blue->>Blue: Tạo CONNECTION_DRAIN (connection_type, count_at_switch = số CLIENT_CONNECTION đang mở)
    par Các connection đang mở tiếp tục chạy trên Blue
        Client->>Blue: WebSocket/long request đang mở tiếp tục nhận phản hồi bình thường
        Blue-->>Client: Hoàn tất connection tự nhiên trên Blue
    and Traffic mới hoàn toàn đi vào Green
        Client->>Green: Mọi request/connection mới được thiết lập với Green
    end
    Blue->>Blue: Theo dõi CONNECTION_DRAIN.drain_status cho tới khi count_at_switch về 0 hoặc hết drain timeout
    alt Toàn bộ connection hoàn tất trước timeout
        Blue->>Blue: drain_status=completed
    else Còn connection kéo dài quá timeout
        Blue->>Blue: drain_status=force_closed_after_timeout (chấp nhận cắt số ít còn lại, không chờ vô hạn)
    end
    Note over Blue: Chỉ sau khi drain_status=completed (hoặc force_closed), Blue mới đủ điều kiện chuyển sang status=retiring và bị tắt hẳn (xem flow "Deploy")
```
