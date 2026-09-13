# Canary deployment cho luồng checkout của sàn e-commerce trong giờ cao điểm

**Hệ thống:** Một sàn e-commerce có luồng checkout tách riêng khỏi các dịch vụ khác (dashboard quản trị, catalog, tìm kiếm), xử lý trực tiếp thanh toán và ảnh hưởng doanh thu theo từng phút.

**Vai trò của flow:** Canary ở đây phải giới hạn phạm vi đúng vào service checkout, tăng tỷ lệ traffic rất thận trọng, và có khả năng phát hiện + rollback trong vài chục giây vì mỗi phút lỗi thanh toán đều mất tiền thật, khác hẳn với canary một dashboard nội bộ ít rủi ro.

**Yêu cầu cụ thể:**
- Giai đoạn canary đầu tiên chỉ nên nhận một tỷ lệ traffic rất nhỏ (ví dụ dưới 1%) và ưu tiên chọn ngẫu nhiên theo request chứ không theo user cố định, để một lỗi ở phiên bản mới không lặp lại nhiều lần trên cùng một khách hàng gây trải nghiệm tệ liên tục.
- Định nghĩa metric rollback riêng cho checkout khác với metric kỹ thuật thông thường — không chỉ tỷ lệ lỗi 5xx mà cả tỷ lệ giao dịch thanh toán thất bại/timeout ở cổng thanh toán, vì phiên bản mới có thể trả về 200 OK nhưng vẫn gọi sai tham số khiến cổng thanh toán từ chối giao dịch.
- Xử lý trường hợp một đơn hàng đã bắt đầu ở phiên bản canary (đã giữ tồn kho, đã tạo intent thanh toán) nhưng canary bị rollback giữa chừng do phát hiện lỗi — đơn hàng đó phải được hoàn tất hoặc hoàn tác một cách nhất quán, không để đơn ở trạng thái treo dở dang khi traffic chuyển hết về phiên bản cũ.
- Đảm bảo cơ chế giám sát tỷ lệ lỗi thanh toán phải tính riêng theo phiên bản (canary vs stable) theo thời gian gần thực (vài giây tới một phút), vì đợi tổng hợp báo cáo theo giờ là quá chậm để tránh thiệt hại doanh thu khi canary có lỗi.
- Thiết kế cơ chế rollback tự động không cần chờ người trực ca xác nhận thủ công đối với các ngưỡng lỗi nghiêm trọng đã định nghĩa trước (ví dụ tỷ lệ thanh toán thất bại tăng gấp đôi so với baseline trong 2 phút), vì tốc độ phản ứng ở đây quan trọng hơn quy trình phê duyệt nhiều bước.

---

# Base ERD — Checkout service trước khi có canary

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có canary deployment. Đề bài nói checkout tách riêng khỏi các service khác, xử lý thanh toán và có đơn hàng gắn tồn kho + payment intent — nên base cần đủ: service, deployment (chỉ 1 target đang chạy), đơn hàng, và payment intent. Chưa có entity nào phục vụ traffic split/metric riêng theo version/rollback policy — những thứ đó là phần enhance.

```mermaid
erDiagram
    CHECKOUT_SERVICE ||--o{ DEPLOYMENT : has
    ORDER ||--o| PAYMENT_INTENT : creates

    CHECKOUT_SERVICE {
        string id PK
        string name "checkout-service"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string status "active"
        datetime deployed_at
    }
    ORDER {
        string id PK
        string status "pending | completed | failed"
        boolean inventory_reserved
        datetime created_at
    }
    PAYMENT_INTENT {
        string id PK
        string order_id FK
        string gateway_status
        datetime created_at
    }
```

---

# Base sequence — Deploy (rolling trực tiếp, chưa có canary)

Đây là **base**, flow "Deploy checkout service" ở trạng thái hiện tại — đẩy thẳng 100% traffic sang bản mới, chỉ giám sát lỗi 5xx chung, rollback thủ công. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm cải tổ lại đúng flow deploy này cho riêng checkout.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Old as Deployment cũ
    participant Router as Router
    participant Dashboard as Monitoring Dashboard (chỉ 5xx chung)
    actor OnCall as On-call Engineer

    CI->>Old: Rolling deploy bản mới, thay thế dần instance cũ
    CI->>Router: Router trỏ 100% traffic sang bản mới ngay khi rolling xong
    Note over Router: Không có giai đoạn traffic nhỏ nào trước khi full 100% — mọi request checkout đều đi qua bản mới ngay lập tức
    Router-->>Dashboard: Traffic chảy qua bản mới
    Dashboard->>Dashboard: Chỉ theo dõi tỷ lệ lỗi 5xx tổng, không tách riêng theo version, không có metric thanh toán riêng
    Note over Dashboard,OnCall: Nếu payment gateway từ chối giao dịch nhưng service vẫn trả 200 OK, dashboard không phát hiện được gì bất thường
    OnCall->>Dashboard: Định kỳ kiểm tra thủ công
    OnCall->>CI: Nếu phát hiện vấn đề, tự tay kích hoạt rollback (mất vài phút)
```

---

# Base sequence — Place order

Đây là **base**, flow "Đặt hàng/checkout" — nền tảng cho enhance: đơn hàng không gắn với version nào đã xử lý nó, nên khi có nhiều version chạy song song (canary), hệ thống chưa có cách xác định và xử lý nhất quán 1 đơn đang dở dang ở đúng version nào.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Service
    participant Inv as Inventory
    participant Gateway as Payment Gateway
    participant DB as ORDER / PAYMENT_INTENT store

    Customer->>App: Xác nhận đặt hàng
    App->>Inv: Giữ tồn kho cho đơn hàng
    Inv-->>App: Giữ thành công
    App->>DB: Tạo ORDER(status=pending, inventory_reserved=true)
    App->>Gateway: Tạo payment intent, thực hiện charge
    Gateway-->>App: Kết quả thanh toán
    App->>DB: Cập nhật ORDER.status theo kết quả (completed/failed)
    App-->>Customer: Kết quả đặt hàng
```

---

# Enhance ERD — sau khi có canary deployment cho checkout

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `DEPLOYMENT` thêm `role` (stable/canary) và `traffic_percent`; `TRAFFIC_SPLIT` (mới) — chia traffic theo request, tăng dần từ tỷ lệ rất nhỏ, thay vì 100% ngay như base.
- `PAYMENT_METRIC_WINDOW` (mới) — đo riêng theo từng version, gần thời gian thực, gồm cả tỷ lệ thanh toán thất bại chứ không chỉ 5xx.
- `ROLLBACK_POLICY` + `ROLLBACK_EVENT` (mới) — tiêu chí rollback tự động cho ngưỡng nghiêm trọng, không chờ xác nhận thủ công.
- `ORDER` thêm `handled_by_deployment_id` — biết chính xác đơn nào đang dở dang ở canary khi rollback xảy ra.
- `IN_FLIGHT_ORDER_RESOLUTION` (mới) — xử lý nhất quán các đơn dở dang trên canary lúc rollback.

```mermaid
erDiagram
    CHECKOUT_SERVICE ||--o{ DEPLOYMENT : has
    CHECKOUT_SERVICE ||--o{ TRAFFIC_SPLIT : configures
    CHECKOUT_SERVICE ||--|| ROLLBACK_POLICY : "governed by"
    DEPLOYMENT ||--o{ PAYMENT_METRIC_WINDOW : measured
    DEPLOYMENT ||--o{ ROLLBACK_EVENT : "may trigger on"
    DEPLOYMENT ||--o{ ORDER : "handled by"
    ORDER ||--o| PAYMENT_INTENT : creates
    ORDER ||--o| IN_FLIGHT_ORDER_RESOLUTION : "resolved via"
    ROLLBACK_EVENT ||--o{ IN_FLIGHT_ORDER_RESOLUTION : triggers

    CHECKOUT_SERVICE {
        string id PK
        string name "checkout-service"
    }
    DEPLOYMENT {
        string id PK
        string service_id FK
        string version
        string role "stable | canary"
        int traffic_percent
        datetime deployed_at
    }
    TRAFFIC_SPLIT {
        string id PK
        string service_id FK
        string canary_deployment_id FK
        string stable_deployment_id FK
        int canary_percent
        string selection_strategy "per_request"
        datetime updated_at
    }
    PAYMENT_METRIC_WINDOW {
        string id PK
        string deployment_id FK
        datetime window_start
        datetime window_end
        decimal payment_failure_rate
        decimal http_5xx_rate
    }
    ROLLBACK_POLICY {
        string id PK
        string service_id FK
        string metric_type "payment_failure_rate | http_5xx_rate"
        string threshold_type "relative_to_baseline | absolute"
        decimal threshold_value
        int window_minutes
        boolean auto_rollback
    }
    ROLLBACK_EVENT {
        string id PK
        string deployment_id FK
        string triggered_by "auto_policy | manual"
        string reason
        datetime triggered_at
    }
    ORDER {
        string id PK
        string status "pending | completed | failed"
        boolean inventory_reserved
        string handled_by_deployment_id FK
        datetime created_at
    }
    PAYMENT_INTENT {
        string id PK
        string order_id FK
        string gateway_status
        datetime created_at
    }
    IN_FLIGHT_ORDER_RESOLUTION {
        string id PK
        string order_id FK
        string rollback_event_id FK
        string resolution "completed_on_canary | compensated_and_retried_on_stable"
        datetime resolved_at
    }
```

---

# Enhance sequence — Deploy (canary theo request, tăng dần)

Đây là **enhance**, cùng flow "Deploy" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 1 của đề bài: canary chỉ nhận dưới 1% traffic ban đầu, chọn ngẫu nhiên theo từng request chứ không cố định theo user, và chỉ tăng dần khi các bước trước đó ổn định.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Stable as Deployment stable (đang chạy)
    participant Canary as Deployment canary (mới)
    participant Split as TRAFFIC_SPLIT store
    participant Router as Router
    actor Customer

    CI->>Canary: Deploy bản mới thành canary, song song với Stable
    CI->>Split: Tạo TRAFFIC_SPLIT(canary_percent=0.5%, selection_strategy=per_request)
    Customer->>Router: Gửi request checkout
    Router->>Split: Random theo từng request (không theo user cố định)
    alt Rơi vào nhóm canary (dưới 1%)
        Router->>Canary: Route request này sang canary
    else Rơi vào nhóm stable (đa số)
        Router->>Stable: Route request này sang stable
    end
    Note over Router,Customer: Cùng 1 khách hàng có thể request lần này vào canary, lần sau vào stable — tránh 1 khách bị dính lỗi canary liên tục nhiều lần
    Note over Split: Sau khi vượt qua bước giám sát (xem flow "Canary monitoring & auto rollback"), canary_percent tăng dần theo bậc (0.5% → 5% → 25% → 100%), mỗi bậc đều được giám sát lại trước khi tăng tiếp
```

---

# Enhance sequence — Canary monitoring & auto rollback

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base chỉ có dashboard 5xx chung, kiểm tra thủ công). Đáp ứng yêu cầu 2, 4 và 5 của đề bài: metric riêng cho thanh toán (không chỉ 5xx), đo theo từng version gần thời gian thực, và rollback tự động cho ngưỡng nghiêm trọng mà không chờ người trực ca.

```mermaid
sequenceDiagram
    participant Canary as Deployment canary
    participant Stable as Deployment stable
    participant Metric as PAYMENT_METRIC_WINDOW store
    participant Monitor as Auto Rollback Monitor
    participant Policy as ROLLBACK_POLICY store
    participant Router as Router / TRAFFIC_SPLIT

    loop Mỗi window ngắn (vài giây tới 1 phút)
        Canary->>Metric: Ghi payment_failure_rate + http_5xx_rate riêng cho canary
        Stable->>Metric: Ghi payment_failure_rate + http_5xx_rate riêng cho stable (làm baseline)
    end

    Monitor->>Metric: Lấy payment_failure_rate của canary và stable trong window gần nhất
    Monitor->>Policy: Lấy threshold_type=relative_to_baseline, threshold_value=2x, window_minutes=2

    alt payment_failure_rate của canary chưa gấp đôi baseline
        Monitor->>Monitor: Tiếp tục theo dõi, cho phép tiến trình tăng traffic ở flow "Deploy"
    else payment_failure_rate của canary vượt ngưỡng nghiêm trọng (gấp đôi baseline trong 2 phút)
        Monitor->>Router: Kích hoạt rollback ngay, không chờ on-call xác nhận
        Router->>Router: Tạo ROLLBACK_EVENT(triggered_by=auto_policy), canary_percent=0
        Router-->>Stable: Toàn bộ traffic quay lại stable
        Monitor-->>Monitor: Gửi cảnh báo cho on-call để điều tra thêm (thông tin, không phải chờ duyệt)
        Note over Router: Đơn hàng đang dở dang trên canary được xử lý ở flow "In-flight order rollback handling"
    end
```

---

# Enhance sequence — In-flight order rollback handling

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm "đơn dở dang trên 1 version cụ thể"). Đáp ứng yêu cầu thứ 3 của đề bài: đơn hàng đã bắt đầu ở canary (đã giữ tồn kho, đã tạo payment intent) nhưng canary bị rollback giữa chừng phải được hoàn tất hoặc hoàn tác nhất quán, không để treo dở dang.

```mermaid
sequenceDiagram
    participant Router as Router (vừa rollback)
    participant Canary as Deployment canary (vẫn đang chạy, chưa bị tắt)
    participant DB as ORDER store
    participant Gateway as Payment Gateway
    participant Inv as Inventory
    participant Resolution as IN_FLIGHT_ORDER_RESOLUTION store

    Note over Router: ROLLBACK_EVENT vừa xảy ra, canary_percent=0, nhưng Canary chưa bị tắt hẳn ngay
    Router->>DB: Tìm toàn bộ ORDER có handled_by_deployment_id=canary và status=pending
    loop Với mỗi đơn dở dang trên canary
        DB->>Canary: Kiểm tra trạng thái xử lý hiện tại của đơn này
        alt Canary vẫn có thể hoàn tất an toàn (đã gọi gateway, đang chờ phản hồi hợp lệ)
            Canary->>Gateway: Cho phép hoàn tất nốt lời gọi đang dở
            Gateway-->>Canary: Kết quả thanh toán
            Canary->>DB: Cập nhật ORDER.status theo kết quả
            Canary->>Resolution: Ghi resolution=completed_on_canary
        else Trạng thái không an toàn để tiếp tục trên canary (vd lỗi/timeout đang xảy ra)
            Canary->>Gateway: Void/huỷ payment intent dở dang (nếu có)
            Canary->>Inv: Nhả lại tồn kho đã giữ
            DB->>DB: Đặt ORDER.status=retry_pending
            Router->>DB: Đơn được retry lại từ đầu trên Stable
            DB->>Resolution: Ghi resolution=compensated_and_retried_on_stable
        end
    end
    Note over DB,Resolution: Không có đơn nào bị bỏ lại ở trạng thái pending vô thời hạn sau khi traffic đã chuyển hết về stable
```
