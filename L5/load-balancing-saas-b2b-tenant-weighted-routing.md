# API gateway route theo tenant có trọng số cho nền tảng SaaS B2B

**Hệ thống:** Một API gateway đứng trước các service backend của nền tảng SaaS B2B, phục vụ nhiều tenant (công ty khách hàng) có quy mô rất khác nhau — một số tenant lớn gửi lượng request gấp hàng trăm lần tenant nhỏ.

**Vai trò của flow:** Load balancing ở đây không chỉ phân phối request đều tới các instance backend, mà còn phải đảm bảo công bằng tài nguyên giữa các tenant — một tenant lớn đột biến traffic không được phép chiếm hết capacity chung khiến các tenant nhỏ hơn bị chậm hoặc timeout.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế giới hạn tài nguyên theo tenant (ví dụ số connection đồng thời hoặc số request/giây tối đa mà một tenant được phép chiếm trên tổng capacity cụm) độc lập với thuật toán chọn instance, để một tenant có traffic bất thường bị giới hạn ở tầng gateway trước khi kịp làm nghẽn backend chung.
- Xử lý trường hợp một tenant lớn đang có traffic đột biến hợp lệ (không phải tấn công, ví dụ họ đang chạy chiến dịch của riêng họ) — phân biệt được với traffic bất thường/lỗi client để không chặn nhầm nhu cầu chính đáng, đồng thời vẫn bảo vệ được tenant khác.
- Đảm bảo việc đếm và giới hạn traffic theo tenant hoạt động đúng khi có nhiều instance gateway chạy song song (bộ đếm phải chia sẻ trạng thái giữa các instance gateway, không phải đếm riêng độc lập từng instance rồi mỗi instance tự cho qua một phần giới hạn dẫn đến tổng vượt ngưỡng thực tế).
- Cho phép cấu hình trọng số route khác nhau theo gói dịch vụ của tenant (tenant trả phí cao hơn được ưu tiên nhiều tài nguyên hơn khi cụm gần đạt tải tối đa) mà không cần phải tách riêng instance backend vật lý cho từng nhóm tenant, tránh lãng phí tài nguyên khi tenant lớn đang rảnh.
- Thiết kế test mô phỏng một tenant gửi traffic tăng đột biến 50 lần trong vài giây, đo độ trễ và tỷ lệ lỗi của các tenant khác trong cùng cụm — phải chứng minh được các tenant khác gần như không bị ảnh hưởng nhờ giới hạn tài nguyên theo tenant đã thiết kế.

---

# Base ERD — API gateway chưa có giới hạn/tách biệt tài nguyên theo tenant

Đây là **base**: trạng thái API gateway *trước khi* áp dụng giới hạn tài nguyên và routing có trọng số theo tenant. Suy luận từ đề bài, base đã có `TENANT` (công ty khách hàng, xác định qua API key/JWT khi gọi request), nhiều `GATEWAY_INSTANCE` chạy song song đứng trước `BACKEND_INSTANCE`, và gateway chọn backend bằng thuật toán đơn giản (round-robin/least-connections) không quan tâm tenant nào đang gửi request. Base chưa có bất kỳ khái niệm quota, bộ đếm chia sẻ, hay trọng số theo gói dịch vụ nào.

```mermaid
erDiagram
    TENANT ||--o{ REQUEST_LOG : "gửi"
    GATEWAY_INSTANCE ||--o{ REQUEST_LOG : "xử lý"
    BACKEND_INSTANCE ||--o{ REQUEST_LOG : "phục vụ"

    TENANT {
        string id PK
        string name
        string plan_tier "free|standard|premium (chưa dùng để routing)"
        string api_key
    }
    GATEWAY_INSTANCE {
        string id PK
        string host
        string status "active|down"
    }
    BACKEND_INSTANCE {
        string id PK
        int active_connections
        string status "active|down"
    }
    REQUEST_LOG {
        string id PK
        string tenant_id FK
        string gateway_instance_id FK
        string backend_instance_id FK
        datetime received_at
        int latency_ms
        string result "success|error|timeout"
    }
```

---

# Base sequence — Route request (round-robin, không phân biệt tenant)

Đây là **base**, flow route request hiện tại: gateway chọn backend instance bằng round-robin/least-connections, hoàn toàn không biết và không quan tâm request đến từ tenant nào. Flow này là tiền đề để so sánh với yêu cầu 1 và yêu cầu 4 của đề bài (giới hạn và trọng số theo tenant).

```mermaid
sequenceDiagram
    actor Tenant as Tenant bất kỳ
    participant GW as Gateway Instance
    participant BE as Backend Instance

    Tenant->>GW: Gửi request kèm API key
    GW->>GW: Xác thực tenant qua API key (chỉ để biết danh tính, không dùng để giới hạn)
    GW->>GW: Chọn backend theo round-robin/least-connections
    GW->>BE: Forward request
    BE-->>GW: Response
    GW-->>Tenant: Trả response
    Note over GW,BE: Không có bước kiểm tra giới hạn tài nguyên nào theo tenant trước khi forward
```

---

# Base sequence — Sự cố 1 tenant lớn làm nghẽn tenant khác (chưa có cô lập tài nguyên)

Đây là **base**, mô tả sự cố xảy ra khi chưa có giới hạn theo tenant: một tenant lớn tăng traffic đột biến, chiếm hết capacity chung của backend, khiến các tenant nhỏ khác cũng bị chậm/timeout dù họ không hề tăng traffic. Đây chính là vấn đề gốc mà toàn bộ đề bài (giới hạn, phân loại traffic, trọng số theo gói) phải giải quyết.

```mermaid
sequenceDiagram
    actor TenantBig as Tenant lớn (traffic tăng đột biến)
    actor TenantSmall as Tenant nhỏ (traffic bình thường)
    participant GW as Gateway Instance
    participant BE as Backend cluster (capacity chung)

    TenantBig->>GW: Gửi lượng request tăng đột biến (không giới hạn)
    GW->>BE: Forward toàn bộ request của Tenant lớn (không kiểm tra quota)
    BE->>BE: Capacity chung gần bão hòa vì request của Tenant lớn
    TenantSmall->>GW: Gửi request bình thường như mọi khi
    GW->>BE: Forward request của Tenant nhỏ
    BE-->>GW: Response chậm/timeout do backend quá tải chung
    GW-->>TenantSmall: Trả lỗi timeout dù Tenant nhỏ không hề tăng traffic
    Note over BE,TenantSmall: Vì không có giới hạn/cô lập theo tenant, 1 tenant lớn có thể chiếm hết capacity chung và ảnh hưởng toàn bộ tenant khác
```

---

# Enhance ERD — Quota theo tenant, bộ đếm chia sẻ, trọng số theo gói dịch vụ

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, thêm mới:

- `TENANT_QUOTA_POLICY` (max_rps, max_concurrent_connections, route_weight theo `plan_tier`) — đáp ứng yêu cầu 1 (giới hạn tài nguyên độc lập với thuật toán chọn instance) và yêu cầu 4 (trọng số route theo gói dịch vụ, không cần tách backend vật lý riêng).
- `RATE_LIMIT_COUNTER` — bộ đếm request/connection theo tenant, đặt trong 1 store chia sẻ (vd Redis) chứ không lưu cục bộ ở từng `GATEWAY_INSTANCE`, đáp ứng yêu cầu 3 (đếm đúng khi có nhiều gateway instance chạy song song).
- `TRAFFIC_SPIKE_EVENT` — ghi nhận mỗi lần phát hiện traffic tăng đột biến của 1 tenant, kèm phân loại (legit_spike/abnormal) và action đã áp dụng, đáp ứng yêu cầu 2 (phân biệt spike hợp lệ với traffic bất thường).
- `BACKEND_INSTANCE.capacity_weight` — dùng cùng `TENANT_QUOTA_POLICY.route_weight` để phân bổ tài nguyên linh hoạt trên cùng 1 cụm backend, không tách instance riêng theo nhóm tenant.

```mermaid
erDiagram
    TENANT ||--o| TENANT_QUOTA_POLICY : "có 1 policy hiện hành"
    TENANT ||--o{ RATE_LIMIT_COUNTER : "có bộ đếm theo từng window"
    TENANT ||--o{ TRAFFIC_SPIKE_EVENT : "có lịch sử spike"
    GATEWAY_INSTANCE ||--o{ REQUEST_LOG : "xử lý"
    BACKEND_INSTANCE ||--o{ REQUEST_LOG : "phục vụ"

    TENANT {
        string id PK
        string name
        string plan_tier "free|standard|premium"
        string api_key
    }
    TENANT_QUOTA_POLICY {
        string id PK
        string tenant_id FK
        int max_rps
        int max_concurrent_connections
        float burst_multiplier "vd 2x cho phép burst tạm thời"
        float route_weight "trọng số ưu tiên theo plan_tier"
        datetime updated_at
    }
    RATE_LIMIT_COUNTER {
        string id PK
        string tenant_id FK
        datetime window_start
        int current_rps "được cộng dồn từ mọi gateway instance"
        int current_connections
    }
    TRAFFIC_SPIKE_EVENT {
        string id PK
        string tenant_id FK
        datetime detected_at
        float spike_ratio "vd 50x baseline"
        string classification "legit_spike|abnormal"
        string action_taken "allow_with_burst|throttle|block"
        string signal_source "vd error_rate thấp + pattern ổn định = legit"
    }
    GATEWAY_INSTANCE {
        string id PK
        string host
        string status "active|down"
    }
    BACKEND_INSTANCE {
        string id PK
        int active_connections
        float capacity_weight
        string status "active|down"
    }
    REQUEST_LOG {
        string id PK
        string tenant_id FK
        string gateway_instance_id FK
        string backend_instance_id FK
        datetime received_at
        int latency_ms
        string result "success|error|timeout|throttled"
    }
```

---

# Enhance sequence — Route request (kiểm tra quota qua bộ đếm chia sẻ trước khi chọn backend)

Đây là **enhance** của flow `route-request` đã có ở base. So với base (chỉ round-robin, không biết tenant), giờ gateway phải: (1) đọc `TENANT_QUOTA_POLICY` để biết giới hạn và trọng số, (2) tăng/kiểm tra `RATE_LIMIT_COUNTER` trong store chia sẻ (không lưu cục bộ) để đếm đúng dù có nhiều gateway instance chạy song song, và (3) chọn backend có tính tới `route_weight` của tenant. Đáp ứng yêu cầu 1, yêu cầu 3, và một phần yêu cầu 4.

```mermaid
sequenceDiagram
    actor Tenant
    participant GW as Gateway Instance (bất kỳ instance nào trong cụm)
    participant SharedCounter as RATE_LIMIT_COUNTER store (Redis, chia sẻ toàn cụm)
    participant Policy as TENANT_QUOTA_POLICY
    participant BE as Backend Instance

    Tenant->>GW: Gửi request kèm API key
    GW->>Policy: Lấy quota hiện hành (max_rps, max_concurrent_connections, route_weight)
    Policy-->>GW: Trả policy của tenant
    GW->>SharedCounter: Atomic increment current_rps cho tenant (đếm chung toàn cụm, không riêng từng instance)
    SharedCounter-->>GW: current_rps sau khi tăng = X
    alt X vượt max_rps (đã tính cả burst_multiplier)
        GW-->>Tenant: 429 Too Many Requests, kèm thông tin quota hiện tại
    else Trong giới hạn quota
        GW->>GW: Chọn backend theo thuật toán chọn instance, có nhân thêm route_weight của tenant
        GW->>BE: Forward request
        BE-->>GW: Response
        GW-->>Tenant: Trả response
    end
    Note over GW,SharedCounter: Vì bộ đếm nằm ở store chia sẻ, dù request của cùng 1 tenant rơi vào các gateway instance khác nhau, tổng vẫn được tính đúng và không vượt ngưỡng thực tế
```

---

# Enhance sequence — Phân loại traffic spike hợp lệ vs bất thường

Đây là **enhance**, flow hoàn toàn mới xử lý khi 1 tenant vượt ngưỡng `max_rps` bình thường. Đáp ứng yêu cầu 2 của đề bài: phân biệt spike hợp lệ (tenant đang chạy chiến dịch riêng của họ) với traffic bất thường/lỗi client, để không chặn nhầm nhu cầu chính đáng nhưng vẫn bảo vệ tenant khác.

```mermaid
sequenceDiagram
    actor Tenant as Tenant đang có traffic tăng đột biến
    participant GW as Gateway Instance
    participant SharedCounter as RATE_LIMIT_COUNTER store
    participant Classifier as Traffic Spike Classifier
    participant SpikeLog as TRAFFIC_SPIKE_EVENT

    Tenant->>GW: Traffic tăng 50x so với baseline
    GW->>SharedCounter: current_rps vượt max_rps thông thường
    GW->>Classifier: Yêu cầu phân loại spike (tenant_id, spike_ratio=50x)
    Classifier->>Classifier: Kiểm tra tín hiệu, error_rate của tenant vẫn thấp, pattern request ổn định, không có dấu hiệu retry storm/lỗi client
    Classifier-->>GW: classification = legit_spike
    GW->>SpikeLog: Ghi TRAFFIC_SPIKE_EVENT (classification=legit_spike, action_taken=allow_with_burst)
    GW->>GW: Cho phép request đi tiếp trong hạn mức burst_multiplier (thay vì chặn cứng ở max_rps)
    GW-->>Tenant: Request được xử lý bình thường trong giới hạn burst tạm thời

    Note over Classifier,SpikeLog: Nếu ngược lại phát hiện error_rate cao bất thường hoặc pattern bất thường (vd retry storm), classification = abnormal, action_taken = throttle/block để bảo vệ tenant khác
```

---

# Enhance sequence — Route có trọng số theo gói dịch vụ khi cụm gần bão hòa

Đây là **enhance**, flow mới tập trung vào yêu cầu 4 của đề bài: khi cụm backend gần đạt tải tối đa, tenant trả phí cao hơn (`plan_tier` = premium) được ưu tiên nhiều tài nguyên hơn qua `route_weight`, mà không cần tách riêng instance backend vật lý cho từng nhóm tenant.

```mermaid
sequenceDiagram
    actor TenantPremium as Tenant Premium
    actor TenantFree as Tenant Free
    participant GW as Gateway Instance
    participant Policy as TENANT_QUOTA_POLICY
    participant LB as Weighted Instance Selector
    participant BE as Backend cluster (chung, gần bão hòa)

    Note over BE: Cụm backend đang ở gần capacity tối đa (vd 90% CPU)
    TenantPremium->>GW: Gửi request
    GW->>Policy: Lấy route_weight của Tenant Premium (vd weight=3)
    TenantFree->>GW: Gửi request cùng thời điểm
    GW->>Policy: Lấy route_weight của Tenant Free (vd weight=1)
    GW->>LB: Đưa cả 2 request vào hàng đợi chọn instance, có trọng số theo route_weight
    LB->>LB: Khi capacity hạn chế, ưu tiên cấp slot xử lý cho request có route_weight cao hơn trước
    LB->>BE: Forward request Tenant Premium trước
    BE-->>GW: Response nhanh cho Tenant Premium
    GW-->>TenantPremium: Trả response, độ trễ thấp
    LB->>BE: Forward request Tenant Free ngay sau khi có slot trống
    BE-->>GW: Response cho Tenant Free (độ trễ cao hơn 1 chút do phải chờ)
    GW-->>TenantFree: Trả response, vẫn thành công nhưng chậm hơn Premium
    Note over LB,BE: Cùng 1 cụm backend vật lý được chia sẻ, không có instance riêng cho từng plan_tier, chỉ khác nhau ở trọng số ưu tiên khi cụm gần bão hòa
```

---

# Enhance sequence — Test 1 tenant tăng traffic 50x, đo ảnh hưởng tới tenant khác

Đây là **enhance**, flow test mô tả ở yêu cầu 5 của đề bài: mô phỏng 1 tenant tăng traffic đột biến 50 lần trong vài giây, đo độ trễ và tỷ lệ lỗi của các tenant khác trong cùng cụm, chứng minh giới hạn tài nguyên theo tenant (yêu cầu 1, 3) đã cô lập được ảnh hưởng. Đối lập trực tiếp với flow base `tenant-overload-incident` — cùng kịch bản traffic nhưng kết quả khác hẳn nhờ enhance.

```mermaid
sequenceDiagram
    participant TestRunner as Test Runner
    actor TenantBig as Tenant lớn (traffic x50 trong vài giây)
    actor TenantSmall as Tenant nhỏ (traffic bình thường, baseline)
    participant GW as Gateway cluster (nhiều instance)
    participant SharedCounter as RATE_LIMIT_COUNTER store
    participant BE as Backend cluster

    TestRunner->>TenantSmall: Ghi nhận baseline latency/error_rate của Tenant nhỏ trước test
    TestRunner->>TenantBig: Bắt đầu bơm traffic tăng 50x trong vài giây
    TenantBig->>GW: Traffic tăng đột biến, rải qua nhiều gateway instance khác nhau
    GW->>SharedCounter: Mỗi request đều tăng bộ đếm chia sẻ của Tenant lớn (tổng đúng dù rải nhiều instance)
    SharedCounter-->>GW: Phần vượt max_rps/burst bị đánh dấu throttled/blocked
    GW-->>TenantBig: Phần lớn traffic vượt ngưỡng bị 429, chỉ phần trong quota được forward
    TenantSmall->>GW: Vẫn gửi traffic bình thường như baseline
    GW->>BE: Forward đầy đủ request của Tenant nhỏ (không bị ảnh hưởng vì Tenant lớn đã bị chặn ở tầng gateway)
    BE-->>GW: Response đúng thời gian như baseline
    GW-->>TenantSmall: Response thành công, latency gần như baseline
    TestRunner->>TestRunner: So sánh latency/error_rate của Tenant nhỏ trước và trong lúc Tenant lớn bị spike
    TestRunner-->>TestRunner: Assert chênh lệch không đáng kể (vd dưới 5%), chứng minh cô lập tài nguyên hoạt động đúng
```
