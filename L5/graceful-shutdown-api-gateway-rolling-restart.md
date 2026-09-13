# API Gateway thực hiện rolling restart toàn cụm backend

**Hệ thống:** Một API Gateway đứng trước nhiều microservice của một nền tảng SaaS, cần rolling restart lần lượt các service trong giờ ít traffic mà không gây downtime cho khách hàng.

**Vai trò của flow:** Gateway phải phối hợp với connection draining của từng service để đảm bảo không có request nào bị route tới instance đang trong quá trình tắt.

**Yêu cầu cụ thể:**
- Gateway phải nhận được tín hiệu "instance X sắp shutdown" trước khi instance đó thực sự ngừng nhận connection (không dựa hoàn toàn vào health check polling định kỳ vốn có độ trễ phát hiện), để loại instance ra khỏi pool routing ngay lập tức.
- Với các kết nối HTTP keep-alive đang mở tới instance sắp tắt, gateway phải đảm bảo request tiếp theo trên kết nối đó (nếu có) được route sang instance khác, không tái sử dụng kết nối cũ tới instance đang chết.
- Trong lúc một service đang rolling restart (một số instance đã restart, một số chưa), phải xử lý được trường hợp version mới và version cũ trả về response có schema khác nhau nhẹ — gateway hoặc client không được crash vì field mới thiếu/thừa.
- Đặt cơ chế circuit breaker tạm thời tăng ngưỡng chịu lỗi (error budget) trong lúc rolling restart đang diễn ra, để tránh gateway hoảng loạn ngắt toàn bộ traffic chỉ vì vài request timeout do đúng lúc instance đang drain.
- Viết test toàn trình: rolling restart lần lượt 5 instance của một service trong lúc có traffic liên tục chạy nền, đo tỷ lệ lỗi/timeout trong toàn bộ quá trình phải bằng 0 hoặc dưới một ngưỡng SLA đã định nghĩa trước.

---

# Base ERD — API Gateway trước khi phối hợp graceful shutdown với backend

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** gateway biết phối hợp connection draining với từng service. Đề bài nói gateway đứng trước nhiều microservice, route request dựa trên trạng thái instance — nên base chỉ cần đủ: các service, các instance của từng service, và trạng thái healthy/unhealthy được cập nhật qua health check định kỳ. Chưa có tín hiệu shutdown chủ động, chưa có theo dõi kết nối keep-alive theo instance, chưa có circuit breaker điều chỉnh động, chưa có khái niệm rolling restart run — tất cả những thứ đó là phần enhance.

```mermaid
erDiagram
    SERVICE ||--o{ INSTANCE : "has"

    SERVICE {
        string id PK
        string name
    }
    INSTANCE {
        string id PK
        string service_id FK
        string host
        int port
        string status "healthy | unhealthy"
        datetime last_health_check_at
    }
```

---

# Base sequence — Route request (chỉ dựa vào health check định kỳ)

Đây là **base**, flow "Gateway route request tới instance" ở trạng thái hiện tại — pool instance chỉ được cập nhật qua health check polling định kỳ (ví dụ mỗi 5 giây), gateway không có cách nào biết instance sắp tắt trước khi health check tự phát hiện. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm rút ngắn/loại bỏ khoảng trễ phát hiện này và các hệ quả của nó.

```mermaid
sequenceDiagram
    actor Client
    participant GW as API Gateway
    participant HC as Health Checker (polling mỗi 5s)
    participant I1 as Instance A (đang chạy bình thường)

    HC->>I1: GET /health (định kỳ)
    I1-->>HC: 200 OK
    HC->>GW: Cập nhật pool: Instance A = healthy

    Note over I1: Vận hành trigger deploy, gửi SIGTERM cho Instance A ngay bây giờ
    I1->>I1: Bắt đầu shutdown, ngừng xử lý request mới ngầm

    Client->>GW: Gửi request
    GW->>I1: Route request (pool vẫn coi Instance A là healthy vì chưa tới lần health check kế tiếp)
    I1-->>GW: Connection reset / timeout (instance đang tắt)
    GW-->>Client: 502 Bad Gateway

    Note over HC,GW: Chỉ khi lần health check định kỳ kế tiếp chạy tới (có thể trễ vài giây) gateway mới biết Instance A unhealthy và loại khỏi pool
    HC->>I1: GET /health (lần kế tiếp)
    I1-->>HC: Không phản hồi
    HC->>GW: Cập nhật pool: Instance A = unhealthy
```

---

# Enhance ERD — Thêm tín hiệu drain, theo dõi connection, circuit breaker động, rolling restart run

Đây là **enhance**, mô hình dữ liệu sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi:
- `INSTANCE` có thêm trạng thái `draining` và cột `schema_version` — đáp ứng yêu cầu 1 (tín hiệu shutdown chủ động) và yêu cầu 3 (nhận biết version khác nhau giữa các instance khi rolling restart đang chạy dở).
- Entity mới `DRAIN_SIGNAL` — sự kiện instance chủ động báo "sắp shutdown" gửi tới gateway trước khi ngừng nhận connection, độc lập với health check polling — đáp ứng yêu cầu 1.
- Entity mới `KEEP_ALIVE_CONNECTION` — theo dõi từng kết nối keep-alive đang mở gắn với instance nào, để gateway biết kết nối nào cần bị đóng/không tái sử dụng khi instance đó drain — đáp ứng yêu cầu 2.
- Entity mới `CIRCUIT_BREAKER` gắn theo service, có `error_budget_multiplier` được nới rộng tạm thời trong lúc rolling restart — đáp ứng yêu cầu 4.
- Entity mới `ROLLING_RESTART_RUN` — ghi nhận 1 lần rolling restart của 1 service (đang restart instance nào, đã xong bao nhiêu, số request/lỗi quan sát được) — làm nền cho bài test toàn trình đo tỷ lệ lỗi — đáp ứng yêu cầu 5.

```mermaid
erDiagram
    SERVICE ||--o{ INSTANCE : "has"
    SERVICE ||--o| CIRCUIT_BREAKER : "protected by"
    SERVICE ||--o{ ROLLING_RESTART_RUN : "restarted via"
    INSTANCE ||--o{ DRAIN_SIGNAL : emits
    INSTANCE ||--o{ KEEP_ALIVE_CONNECTION : serves
    ROLLING_RESTART_RUN ||--o{ DRAIN_SIGNAL : includes

    SERVICE {
        string id PK
        string name
    }
    INSTANCE {
        string id PK
        string service_id FK
        string host
        int port
        string status "healthy | draining | unhealthy"
        string schema_version
        datetime last_health_check_at
    }
    DRAIN_SIGNAL {
        string id PK
        string instance_id FK
        string reason "sigterm | rolling_restart"
        datetime signaled_at
        datetime deregistered_at
    }
    KEEP_ALIVE_CONNECTION {
        string id PK
        string instance_id FK
        string client_id
        string status "open | closing | closed"
        datetime established_at
    }
    CIRCUIT_BREAKER {
        string id PK
        string service_id FK
        string state "closed | open | half_open"
        float error_threshold
        float error_budget_multiplier "1.0 mac dinh, tang tam thoi khi rolling restart"
        datetime widened_until
    }
    ROLLING_RESTART_RUN {
        string id PK
        string service_id FK
        int total_instances
        int instances_restarted
        int request_count
        int error_count
        float sla_error_threshold
        string status "running | completed | failed"
    }
```

---

# Enhance sequence — Route request (drain signal chủ động, không tái dùng keep-alive cũ)

Đây là **enhance** của flow `route-request` đã có ở base, đáp ứng **yêu cầu 1 và 2** của đề bài. So với base: instance chủ động gửi `DRAIN_SIGNAL` cho gateway ngay khi nhận SIGTERM (không đợi health check polling phát hiện), gateway loại instance khỏi pool routing ngay lập tức. Với các kết nối HTTP keep-alive đang mở tới instance đó, gateway đóng kết nối (hoặc đánh dấu closing) để request tiếp theo trên client bắt buộc mở kết nối mới tới instance khác, không bao giờ tái sử dụng kết nối cũ tới instance đang chết.

```mermaid
sequenceDiagram
    actor Client
    participant GW as API Gateway
    participant I1 as Instance A (sắp shutdown)
    participant I2 as Instance B (còn khỏe)

    Client->>GW: Request 1 (mở keep-alive connection C1)
    GW->>I1: Route qua C1
    I1-->>GW: 200 OK
    GW-->>Client: 200 OK (C1 vẫn giữ mở)

    Note over I1: Vận hành gửi SIGTERM cho Instance A
    I1->>GW: DRAIN_SIGNAL(instance=A, reason=sigterm) -- chủ động, không đợi health check
    GW->>GW: Đánh dấu Instance A = draining, loại khỏi pool routing ngay
    GW->>I1: Đóng kết nối keep-alive C1 (status=closing)

    Client->>GW: Request 2 trên C1 (client tưởng vẫn dùng lại kết nối cũ)
    GW-->>Client: Báo C1 đã đóng, buộc mở kết nối mới
    Client->>GW: Request 2 (mở keep-alive connection C2 mới)
    GW->>I2: Route qua C2 (Instance A đã bị loại khỏi pool)
    I2-->>GW: 200 OK
    GW-->>Client: 200 OK

    I1->>I1: Hoàn tất drain các request đang xử lý dở, sau đó tắt hẳn
```

---

# Enhance sequence — Chịu đựng schema khác nhau giữa version cũ và mới khi rolling restart dở dang

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 3** của đề bài: trong lúc rolling restart, một phần instance đã lên version mới (schema response có thêm/bớt field), phần còn lại vẫn version cũ — gateway và client không được crash vì sự khác biệt này.

```mermaid
sequenceDiagram
    actor Client
    participant GW as API Gateway
    participant Iold as Instance (version cũ, chưa restart)
    participant Inew as Instance (version mới, đã restart)

    Client->>GW: Request A
    GW->>Iold: Route request
    Iold-->>GW: 200 OK { order_id, status, total } (schema cũ, không có field discount_code)
    GW->>GW: Parse response bằng schema tolerant, field thiếu discount_code lấy giá trị mặc định null, không lỗi
    GW-->>Client: 200 OK { order_id, status, total, discount_code: null }

    Client->>GW: Request B
    GW->>Inew: Route request
    Inew-->>GW: 200 OK { order_id, status, total, discount_code, loyalty_points } (schema mới, thêm 2 field)
    GW->>GW: Parse response bằng schema tolerant, field lạ loyalty_points được giữ nguyên hoặc bỏ qua tuỳ hợp đồng API, không crash vì field thừa
    GW-->>Client: 200 OK { order_id, status, total, discount_code, loyalty_points }

    Note over GW,Inew: Gateway dùng chế độ parse "unknown fields allowed, missing optional fields = default" trong suốt thời gian INSTANCE.status của cả 2 version cùng tồn tại trong pool
```

---

# Enhance sequence — Nới error budget circuit breaker tạm thời trong lúc rolling restart

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 4** của đề bài: khi rolling restart đang diễn ra, một vài request timeout do đúng lúc instance đang drain là chuyện bình thường và có thể dự đoán trước — circuit breaker phải tạm nới ngưỡng chịu lỗi trong cửa sổ thời gian đó, tránh ngắt toàn bộ traffic của service chỉ vì vài lỗi tạm thời.

```mermaid
sequenceDiagram
    actor Ops as Đội vận hành
    participant GW as API Gateway
    participant CB as Circuit Breaker (service X)
    participant RR as ROLLING_RESTART_RUN

    Ops->>RR: Bắt đầu rolling restart service X
    RR->>CB: Thông báo rolling restart đang chạy
    CB->>CB: UPDATE error_budget_multiplier = 3.0, widened_until = now + 10 phút
    Note over CB: Ngưỡng lỗi cho phép trong cửa sổ này tăng gấp 3 lần mức bình thường, chỉ áp dụng tạm thời

    loop Trong lúc rolling restart
        GW->>CB: Ghi nhận kết quả mỗi request (success/timeout)
        alt Tỷ lệ lỗi vẫn dưới ngưỡng đã nới rộng
            CB->>CB: state vẫn = closed, tiếp tục cho traffic đi qua bình thường
        else Tỷ lệ lỗi vượt cả ngưỡng đã nới rộng (có bug thật, không phải do drain)
            CB->>CB: state = open, ngắt traffic tới service X
            CB-->>GW: Từ chối route request mới tới service X
        end
    end

    RR->>CB: Rolling restart hoàn tất
    CB->>CB: UPDATE error_budget_multiplier = 1.0 (khôi phục ngưỡng bình thường)
```

---

# Enhance sequence — Test toàn trình rolling restart 5 instance dưới traffic liên tục

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: kịch bản test tự động rolling restart lần lượt 5 instance của một service trong lúc có traffic liên tục chạy nền, đo tỷ lệ lỗi/timeout toàn bộ quá trình và so sánh với ngưỡng SLA đã định nghĩa trước.

```mermaid
sequenceDiagram
    participant Test as Test Orchestrator
    participant LoadGen as Load Generator (traffic nền)
    participant GW as API Gateway
    participant RR as ROLLING_RESTART_RUN
    participant Cluster as Cluster 5 instance (Service X)

    Test->>RR: INSERT ROLLING_RESTART_RUN(total_instances=5, status=running)
    Test->>LoadGen: Bắt đầu bắn traffic liên tục qua GW
    LoadGen->>GW: Request liên tục trong suốt quá trình test

    loop Với từng instance trong 5 instance (tuần tự)
        Test->>Cluster: Gửi SIGTERM cho instance kế tiếp
        Cluster->>GW: DRAIN_SIGNAL(instance=kế tiếp)
        GW->>GW: Loại instance khỏi pool, đóng keep-alive cũ, route tiếp qua 4 instance còn lại
        Cluster->>Cluster: Instance restart xong, đăng ký lại vào pool (status=healthy)
        Test->>RR: UPDATE instances_restarted += 1
    end

    LoadGen->>Test: Dừng traffic, tổng hợp request_count và error_count (timeout/5xx)
    Test->>RR: UPDATE request_count, error_count
    Test->>Test: Tính tỷ lệ lỗi = error_count / request_count

    alt Tỷ lệ lỗi <= sla_error_threshold đã định nghĩa trước
        Test->>RR: UPDATE status=completed
        Test-->>Test: Test PASS
    else Tỷ lệ lỗi vượt ngưỡng SLA
        Test->>RR: UPDATE status=failed
        Test-->>Test: Test FAIL, đính kèm log các request lỗi để điều tra
    end
```
