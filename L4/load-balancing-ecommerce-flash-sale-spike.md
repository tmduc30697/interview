# E-commerce chịu tải đột biến trong flash sale

**Hệ thống:** Một sàn e-commerce có checkout service, trong sự kiện flash sale traffic có thể tăng 20 lần trong vài giây.

**Vai trò của flow:** Load balancer phải tự chuyển đổi hành vi giữa các thuật toán (round-robin lúc bình thường, least-connection khi có instance chạy job nặng) và bảo vệ backend khỏi bị sập toàn cụm.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế phát hiện một instance "chậm" (latency tăng cao) dù vẫn healthy theo health check thông thường, và giảm trọng số route tới instance đó (outlier detection), tránh trường hợp health check pass nhưng response time đã tệ.
- Khi toàn bộ cluster đạt ngưỡng tải tối đa, phải có cơ chế shed load có kiểm soát (trả lỗi 503 kèm `Retry-After` cho một phần request) thay vì để tất cả request timeout đồng loạt.
- Đảm bảo thuật toán LB không route request "mua hàng" (ghi dữ liệu, cần nhất quán) và request "xem sản phẩm" (đọc, có thể cache) theo cùng một cách — phân tách được pool instance hoặc trọng số theo loại traffic.
- Test race condition: nhiều request cùng cập nhật bộ đếm connection/tải của một instance đồng thời — đảm bảo số liệu dùng để ra quyết định route không bị lệch do cập nhật không atomic.
- Có cơ chế circuit breaker phối hợp với LB: khi một instance liên tục trả lỗi 5xx, tự động loại khỏi vòng quay trong một khoảng thời gian rồi thử lại dần (half-open), không chờ health check thủ công.

---

# Base ERD — Load balancer checkout trước khi có outlier detection, shedding, pool tách biệt và circuit breaker

Đây là **base**: mô hình dữ liệu suy luận cho load balancer đứng trước checkout service của một sàn e-commerce, ở trạng thái **trước khi** áp các yêu cầu về outlier detection, shed load, tách pool theo loại traffic, chống race condition đếm tải, và circuit breaker phối hợp. Base chỉ cần đủ: cấu hình LB có thể tự chuyển giữa round-robin và least-connection (đúng như mô tả vai trò hiện tại của flow), một pool instance duy nhất phục vụ mọi loại request, và health check nhị phân đơn giản — đủ để các yêu cầu enhance "có nghĩa" khi so sánh.

```mermaid
erDiagram
    LOAD_BALANCER_CONFIG ||--o{ INSTANCE : "điều phối"
    INSTANCE ||--o{ REQUEST : "phục vụ"

    LOAD_BALANCER_CONFIG {
        string id PK
        string current_algorithm "round_robin | least_connection"
        datetime updated_at
    }
    INSTANCE {
        string id PK
        string host
        string health_status "healthy | unhealthy"
        int active_connections
        boolean running_heavy_job
    }
    REQUEST {
        string id PK
        string type "checkout | view_product"
        string instance_id FK
        datetime routed_at
        int response_status
    }
```

---

# Base sequence — Request routing

Đây là **base**, flow "Định tuyến request" ở trạng thái hiện tại: LB tự chuyển giữa round-robin (bình thường) và least-connection (khi có instance đang chạy job nặng), nhưng đối xử với mọi loại request (xem sản phẩm lẫn mua hàng) hoàn toàn như nhau, và đọc/ghi `active_connections` một cách đơn giản không đảm bảo atomic. Flow này là tiền đề cho **yêu cầu 3** (tách pool/trọng số theo loại traffic) và **yêu cầu 4** (race condition khi cập nhật bộ đếm tải) vì cả hai đều tác động trực tiếp lên đúng bước ra quyết định này.

```mermaid
sequenceDiagram
    actor CustomerView as Customer xem sản phẩm
    actor CustomerBuy as Customer bấm mua hàng
    participant LB as Load Balancer
    participant InstanceA as Instance A (đang chạy job nặng)
    participant InstanceB as Instance B (rảnh)

    CustomerView->>LB: Request xem sản phẩm
    LB->>LB: Kiểm tra current_algorithm, thấy có instance chạy job nặng nên dùng least_connection
    LB->>LB: Đọc active_connections của từng instance (đọc rồi cộng, không atomic)
    LB-->>CustomerView: Route tới InstanceB (ít connection hơn)

    CustomerBuy->>LB: Request mua hàng, gần như cùng lúc
    LB->>LB: Áp dụng đúng cùng thuật toán least_connection như request xem sản phẩm, không phân biệt loại traffic
    LB-->>CustomerBuy: Route tới InstanceB (cùng pool, cùng cách tính)
    Note over LB,InstanceB: Request đọc (xem sản phẩm) và request ghi (mua hàng, cần nhất quán) bị route lẫn lộn vào cùng instance theo cùng một cách, không có pool hay trọng số riêng
```

---

# Base sequence — Traffic spike handling

Đây là **base**, flow "Xử lý khi traffic tăng đột biến" ở trạng thái hiện tại: khi flash sale khiến traffic tăng 20 lần, LB tiếp tục route mọi request như bình thường, không có outlier detection, không shed load, không circuit breaker — cả cụm bị dồn tải tới khi sập. Flow này là tiền đề cho **yêu cầu 1** (outlier detection), **yêu cầu 2** (shed load có kiểm soát) và **yêu cầu 5** (circuit breaker) vì cả ba đều nhằm xử lý đúng tình huống này.

```mermaid
sequenceDiagram
    actor Customers as Hàng loạt customer (traffic x20)
    participant LB as Load Balancer
    participant InstanceA as Instance A (latency tăng cao nhưng health check vẫn pass)
    participant InstanceB as Instance B (bắt đầu trả 5xx liên tục)
    participant InstanceC as Instance C

    Customers->>LB: Request đồng loạt tăng vọt
    LB->>InstanceA: Health check định kỳ
    InstanceA-->>LB: OK (health check pass dù response time đã tệ)
    LB->>InstanceB: Health check định kỳ
    InstanceB-->>LB: OK (health check pass dù đang trả 5xx cho request thật)
    LB->>LB: Vẫn route đều theo thuật toán hiện tại cho cả InstanceA, InstanceB, InstanceC
    LB-->>Customers: Một phần request rơi vào InstanceA (chậm) hoặc InstanceB (lỗi 5xx)
    Note over LB,InstanceC: Khi cả cụm đạt ngưỡng tải tối đa, không có cơ chế shed load nào, toàn bộ request tiếp tục dồn vào cho tới khi timeout đồng loạt
```

---

# Enhance ERD — sau khi bổ sung outlier detection, load shedding, tách pool, chống race condition và circuit breaker

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có các entity/field mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `INSTANCE` (sửa) — thêm `latency_p95_ms`, `routing_weight`, `consecutive_5xx_count`, `circuit_state`, `connection_count_version` (yêu cầu 1, 4, 5).
- `OUTLIER_SAMPLE` (mới) — mẫu latency đo định kỳ từng instance, dùng phát hiện instance chậm dù health check vẫn pass (yêu cầu 1).
- `POOL_ROUTING_WEIGHT` (mới) — trọng số route riêng theo từng loại traffic (buy/view) cho từng instance, tách biệt cách đối xử giữa request đọc và ghi (yêu cầu 3).
- `LOAD_SHED_DECISION` (mới) — ghi nhận quyết định shed load (trả 503 kèm Retry-After) khi cả cụm đạt ngưỡng tải, thay vì để tất cả cùng timeout (yêu cầu 2).
- `CONNECTION_COUNTER_UPDATE` (mới) — log mọi lần cập nhật `active_connections`, dùng `resulting_version` để đảm bảo cập nhật atomic, tránh race condition (yêu cầu 4).
- `CIRCUIT_BREAKER_EVENT` (mới) — ghi lại mỗi lần instance bị loại khỏi vòng quay do liên tục lỗi 5xx và các lần thử lại half-open (yêu cầu 5).

```mermaid
erDiagram
    LOAD_BALANCER_CONFIG ||--o{ INSTANCE : "điều phối"
    INSTANCE ||--o{ REQUEST : "phục vụ"
    INSTANCE ||--o{ OUTLIER_SAMPLE : "được đo latency"
    INSTANCE ||--o{ POOL_ROUTING_WEIGHT : "có trọng số theo loại traffic"
    REQUEST ||--o| LOAD_SHED_DECISION : "có thể bị shed"
    INSTANCE ||--o{ CONNECTION_COUNTER_UPDATE : "nhận cập nhật atomic"
    INSTANCE ||--o{ CIRCUIT_BREAKER_EVENT : "có lịch sử chuyển trạng thái"

    LOAD_BALANCER_CONFIG {
        string id PK
        string current_algorithm "round_robin | least_connection"
        datetime updated_at
    }
    INSTANCE {
        string id PK
        string host
        string health_status "healthy | unhealthy"
        int active_connections
        boolean running_heavy_job
        int latency_p95_ms
        decimal routing_weight
        int consecutive_5xx_count
        string circuit_state "closed | open | half_open"
        int connection_count_version
    }
    REQUEST {
        string id PK
        string type "checkout | view_product"
        string instance_id FK
        datetime routed_at
        int response_status
    }
    OUTLIER_SAMPLE {
        string id PK
        string instance_id FK
        int latency_ms
        datetime sampled_at
    }
    POOL_ROUTING_WEIGHT {
        string instance_id PK, FK
        string pool_type PK "checkout_write | view_product_read"
        decimal weight
    }
    LOAD_SHED_DECISION {
        string id PK
        string request_id FK
        string decision "shed | serve"
        int retry_after_seconds
        decimal cluster_load_snapshot
        datetime decided_at
    }
    CONNECTION_COUNTER_UPDATE {
        string id PK
        string instance_id FK
        int delta
        int resulting_version
        string source_request_id FK
        datetime recorded_at
    }
    CIRCUIT_BREAKER_EVENT {
        string id PK
        string instance_id FK
        string from_state "closed | open | half_open"
        string to_state "closed | open | half_open"
        string reason
        datetime occurred_at
    }
```

---

# Enhance sequence — Request routing (tách pool theo loại traffic)

Đây là **enhance** của flow đã có ở base "Request routing". So với base, LB tra `POOL_ROUTING_WEIGHT` để đối xử khác nhau giữa request đọc (xem sản phẩm, có thể cache, ưu tiên rải đều/ít quan tâm nhất quán) và request ghi (mua hàng, cần nhất quán, ưu tiên instance ổn định), đáp ứng **yêu cầu 3** (không route request "mua hàng" và "xem sản phẩm" theo cùng một cách).

```mermaid
sequenceDiagram
    actor CustomerView as Customer xem sản phẩm
    actor CustomerBuy as Customer bấm mua hàng
    participant LB as Load Balancer
    participant Pool as POOL_ROUTING_WEIGHT
    participant InstanceA as Instance A (ổn định, ưu tiên cho buy)
    participant InstanceB as Instance B (chạy job nặng, chỉ nhận view)

    CustomerView->>LB: Request xem sản phẩm (type=view_product)
    LB->>Pool: Tra trọng số pool_type=view_product_read cho từng instance
    Pool-->>LB: InstanceB vẫn nhận được trọng số view hợp lý dù đang chạy job nặng, vì request đọc chấp nhận cache/độ trễ nhẹ
    LB-->>CustomerView: Route tới InstanceB

    CustomerBuy->>LB: Request mua hàng (type=checkout), gần như cùng lúc
    LB->>Pool: Tra trọng số pool_type=checkout_write cho từng instance
    Pool-->>LB: InstanceA có trọng số checkout_write cao hơn vì đang ổn định, cần nhất quán
    LB-->>CustomerBuy: Route tới InstanceA, tách biệt hẳn khỏi pool phục vụ view_product
    Note over LB,Pool: Cùng một cụm instance nhưng hai loại traffic được cân trọng số độc lập, tránh request ghi bị ảnh hưởng bởi tải của request đọc và ngược lại
```

---

# Enhance sequence — Traffic spike handling (shed load có kiểm soát)

Đây là **enhance** của flow đã có ở base "Traffic spike handling". So với base, khi toàn cụm đạt ngưỡng tải tối đa, LB chủ động trả lỗi 503 kèm `Retry-After` cho một phần request thay vì để tất cả cùng dồn vào và timeout đồng loạt, đáp ứng **yêu cầu 2** (cơ chế shed load có kiểm soát khi cluster đạt ngưỡng tải tối đa).

```mermaid
sequenceDiagram
    actor Customers as Hàng loạt customer (traffic x20)
    participant LB as Load Balancer
    participant ShedController as Load Shed Controller
    participant Cluster as Toàn bộ Instance trong cụm

    Customers->>LB: Request đồng loạt tăng vọt
    LB->>ShedController: Kiểm tra cluster_load_snapshot hiện tại
    ShedController->>Cluster: Tổng hợp active_connections + latency toàn cụm
    Cluster-->>ShedController: Cụm đã đạt ngưỡng tải tối đa

    alt Request thuộc phần vượt ngưỡng chịu tải an toàn
        ShedController->>ShedController: Ghi LOAD_SHED_DECISION(decision=shed, retry_after_seconds=N)
        ShedController-->>LB: Yêu cầu từ chối request này
        LB-->>Customers: Trả 503 kèm Retry-After: N giây, không forward vào cụm
    else Request nằm trong ngưỡng cụm còn xử lý được
        ShedController->>ShedController: Ghi LOAD_SHED_DECISION(decision=serve)
        LB->>Cluster: Forward request bình thường
        Cluster-->>Customers: Xử lý và trả kết quả
    end
    Note over ShedController,Cluster: Một phần request bị từ chối có kiểm soát giúp phần còn lại vẫn được xử lý đúng hạn, thay vì toàn bộ cùng timeout
```

---

# Enhance sequence — Outlier detection

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chỉ có health check nhị phân, không phát hiện được instance chậm. Đáp ứng **yêu cầu 1** (phát hiện instance "chậm" dù vẫn healthy theo health check thông thường, giảm trọng số route tới instance đó).

```mermaid
sequenceDiagram
    participant InstanceA as Instance A (latency tăng cao)
    participant HealthCheck as Health Check định kỳ
    participant OutlierDetector as Outlier Detector
    participant LB as Load Balancer

    HealthCheck->>InstanceA: Ping kiểm tra sống/chết
    InstanceA-->>HealthCheck: OK (health check pass bình thường)

    loop Liên tục đo theo từng request thật
        InstanceA->>OutlierDetector: Ghi OUTLIER_SAMPLE(latency_ms của response vừa xử lý)
    end
    OutlierDetector->>OutlierDetector: Tính latency_p95_ms của InstanceA trong cửa sổ gần nhất
    alt latency_p95_ms vượt ngưỡng so với các instance khác trong cụm
        OutlierDetector->>LB: Yêu cầu giảm routing_weight của InstanceA
        LB->>LB: Giảm dần tỉ lệ request route tới InstanceA, dù health check vẫn pass
        Note over LB,InstanceA: Response time tệ được phát hiện và xử lý sớm, không chờ tới khi health check phát hiện lỗi hệ thống
    else latency bình thường
        OutlierDetector->>LB: Giữ nguyên routing_weight
    end
```

---

# Enhance sequence — Connection counter race condition

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base đọc/ghi `active_connections` một cách đơn giản không đảm bảo atomic. Đáp ứng **yêu cầu 4** (nhiều request cùng cập nhật bộ đếm connection/tải của một instance đồng thời, đảm bảo số liệu dùng để ra quyết định route không bị lệch do cập nhật không atomic).

```mermaid
sequenceDiagram
    actor RequestX as Request X
    actor RequestY as Request Y (gần như đồng thời)
    participant LB as Load Balancer
    participant Counter as CONNECTION_COUNTER_UPDATE (atomic, có version)
    participant InstanceA as Instance A

    par Hai request tới gần như cùng lúc
        RequestX->>LB: Route tới InstanceA
        LB->>Counter: Ghi delta=+1 (atomic increment), lấy resulting_version
        Counter-->>LB: active_connections cập nhật thành N+1, version tăng đúng 1 nấc
    and
        RequestY->>LB: Route tới InstanceA
        LB->>Counter: Ghi delta=+1 (atomic increment), lấy resulting_version
        Counter-->>LB: active_connections cập nhật thành N+2, version tăng đúng 1 nấc tiếp theo
    end
    Note over Counter,InstanceA: Nhờ increment atomic thay vì đọc-cộng-ghi riêng lẻ, active_connections phản ánh đúng N+2, không bị mất một trong hai lần cập nhật

    RequestX->>InstanceA: Xử lý xong, trả kết quả
    LB->>Counter: Ghi delta=-1 (atomic decrement)
    Counter-->>LB: active_connections giảm đúng về N+1
    Note over LB,Counter: Quyết định route tiếp theo dựa trên số liệu chính xác, không bị lệch do race condition giữa các lần cập nhật đồng thời
```

---

# Enhance sequence — Circuit breaker phối hợp với LB

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chỉ dựa vào health check thủ công để loại instance lỗi. Đáp ứng **yêu cầu 5** (circuit breaker phối hợp với LB: instance liên tục trả lỗi 5xx tự động bị loại khỏi vòng quay trong một khoảng thời gian rồi thử lại dần qua half-open, không chờ health check thủ công).

```mermaid
sequenceDiagram
    actor Customers
    participant LB as Load Balancer
    participant InstanceB as Instance B (bắt đầu lỗi 5xx liên tục)
    participant Breaker as Circuit Breaker (theo từng instance)

    loop Một loạt request liên tiếp
        Customers->>LB: Request
        LB->>InstanceB: Forward request (circuit_state=closed)
        InstanceB-->>LB: Trả 5xx
        LB->>Breaker: Ghi nhận lỗi, tăng consecutive_5xx_count
    end

    alt consecutive_5xx_count vượt ngưỡng
        Breaker->>Breaker: Chuyển circuit_state=open, ghi CIRCUIT_BREAKER_EVENT(from=closed, to=open)
        Breaker->>LB: Yêu cầu loại InstanceB khỏi vòng quay ngay lập tức
        LB-->>Customers: Không route request nào tới InstanceB nữa trong thời gian breaker mở
    end

    Note over Breaker: Sau một khoảng thời gian cố định
    Breaker->>Breaker: Chuyển circuit_state=half_open, ghi CIRCUIT_BREAKER_EVENT(from=open, to=half_open)
    LB->>InstanceB: Cho phép một lượng nhỏ request thử nghiệm đi qua
    InstanceB-->>LB: Kết quả request thử

    alt Request thử nghiệm thành công
        Breaker->>Breaker: Chuyển circuit_state=closed, reset consecutive_5xx_count
        LB->>InstanceB: Route lại bình thường
    else Vẫn còn lỗi 5xx
        Breaker->>Breaker: Quay lại circuit_state=open, gia hạn thời gian mở
    end
```
