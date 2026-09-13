# Truy vết một đơn hàng e-commerce qua giỏ hàng, tồn kho, thanh toán và xác nhận

**Hệ thống:** Một sàn e-commerce có luồng đặt hàng đi qua bốn service tách biệt: `cart-service` → `inventory-service` (kiểm tra và giữ tồn kho) → `payment-service` (gọi cổng thanh toán bên ngoài) → `order-confirmation-service`, mỗi service có tài nguyên và tốc độ scale khác nhau.

**Vai trò của flow:** Khi checkout bị chậm vào giờ cao điểm, tracing phải chỉ ra chính xác bước nào đang là nút thắt (tồn kho đang bị lock tranh chấp, hay cổng thanh toán bên ngoài đang chậm, hay chỉ đơn giản là hàng đợi nội bộ đang nghẽn) để đội vận hành không phải đoán mò giữa nhiều nguyên nhân khả dĩ.

**Yêu cầu cụ thể:**
- Trace của một đơn hàng phải giữ nguyên `trace_id` xuyên suốt bốn bước dù chúng chạy trên các service độc lập với vòng đời request khác nhau (một số đồng bộ, một số phải chờ callback bất đồng bộ từ cổng thanh toán), để dựng lại được toàn bộ hành trình của một đơn hàng cụ thể khi khách hàng báo lỗi.
- Bước kiểm tra tồn kho thường liên quan tới việc chờ lock (nhiều đơn hàng cùng tranh chấp một sản phẩm hot), span của bước này phải tách riêng thời gian chờ lock khỏi thời gian xử lý logic thật, để phân biệt được "chậm vì tranh chấp tồn kho" với "chậm vì code xử lý nặng".
- Bước gọi cổng thanh toán bên ngoài phải được đánh dấu rõ là external dependency với SLA riêng, để khi tổng hợp thống kê "checkout chậm ở bước nào" vào giờ cao điểm, đội vận hành không nhầm lẫn giữa độ trễ do chính hệ thống gây ra và độ trễ nằm ngoài tầm kiểm soát của cổng thanh toán đối tác.
- Thiết kế dashboard tổng hợp theo từng bước trên toàn bộ traffic (không chỉ theo từng đơn hàng riêng lẻ) để trả lời câu hỏi "giờ cao điểm hôm nay chậm hơn hôm qua, chậm chủ yếu ở bước nào" — cho phép so sánh phân phối độ trễ (p50/p95/p99) từng bước theo thời gian thực.
- Xử lý trường hợp một đơn hàng bị timeout ở bước thanh toán nhưng cổng thanh toán thực ra đã xử lý thành công phía họ và gửi callback trễ — trace phải ghi lại đủ để đối chiếu, tránh tình huống hệ thống báo "đơn hàng thất bại" trong khi thực tế tiền đã bị trừ, gây khiếu nại khó điều tra nếu thiếu dữ liệu trace đầy đủ.

---

# ERD - Base (trước khi có distributed tracing)

Đây là trạng thái **base**: sàn e-commerce đã có luồng đặt hàng đi qua 4 service tách biệt (`cart-service`, `inventory-service`, `payment-service`, `order-confirmation-service`), có các entity nghiệp vụ cốt lõi là `ORDER`, `INVENTORY_HOLD` (giữ tồn kho) và `PAYMENT_TRANSACTION`, nhưng mỗi service chỉ ghi log cục bộ, không có định danh chung nối các bước lại với nhau. Đây là tiền đề khiến enhance cần thêm tracing để chỉ ra chính xác bước nào đang là nút thắt.

```mermaid
erDiagram
    ORDER ||--o| INVENTORY_HOLD : "giữ tồn kho cho"
    ORDER ||--o| PAYMENT_TRANSACTION : "thanh toán qua"
    ORDER ||--o{ SERVICE_LOG : "phát sinh log rải rác"

    ORDER {
        string order_id PK
        string cart_id
        string status
        datetime created_at
    }
    INVENTORY_HOLD {
        string hold_id PK
        string order_id FK
        string sku
        int quantity
        string status
    }
    PAYMENT_TRANSACTION {
        string payment_id PK
        string order_id FK
        decimal amount
        string gateway_reference
        string status
    }
    SERVICE_LOG {
        string log_id PK
        string service_name
        string level
        string message
        datetime logged_at
    }
```

---

# Sequence Diagram - Base: checkout-order

Đây là flow **base**: đơn hàng đi qua `cart-service` -> `inventory-service` -> `payment-service` (gọi cổng thanh toán bên ngoài) -> `order-confirmation-service`, mỗi service chỉ ghi log cục bộ của riêng nó. Flow này là nền để so sánh với enhance, nơi mỗi bước sẽ được gắn `trace_id`/`span` xuyên suốt kể cả đoạn chờ callback bất đồng bộ.

```mermaid
sequenceDiagram
    actor Customer
    participant Cart as cart-service
    participant Inv as inventory-service
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài
    participant Confirm as order-confirmation-service

    Customer->>Cart: Checkout giỏ hàng
    Cart->>Cart: Ghi log cục bộ "tạo order"
    Cart->>Inv: Yêu cầu giữ tồn kho cho order
    Inv->>Inv: Ghi log cục bộ "giữ tồn kho"
    Inv-->>Cart: Giữ tồn kho thành công
    Cart->>Pay: Yêu cầu thanh toán
    Pay->>Gateway: Gọi cổng thanh toán bên ngoài
    Gateway-->>Pay: Thanh toán thành công
    Pay->>Pay: Ghi log cục bộ "thanh toán thành công"
    Pay-->>Cart: Thanh toán OK
    Cart->>Confirm: Yêu cầu xác nhận đơn hàng
    Confirm->>Confirm: Ghi log cục bộ "xác nhận đơn hàng"
    Confirm-->>Customer: Đơn hàng đã xác nhận
```

---

# Sequence Diagram - Base: diagnose-slow-checkout

Đây là flow **base** minh hoạ chính vấn đề mà đề bài cần giải quyết: vào giờ cao điểm, checkout bị chậm nhưng đội vận hành không có cách nào xác định chính xác nguyên nhân - có thể do tồn kho bị lock tranh chấp, cổng thanh toán bên ngoài chậm, hay chỉ đơn giản hàng đợi nội bộ nghẽn - vì log của từng service không liên kết với nhau và không tách được thời gian chờ khỏi thời gian xử lý thật. Đây là lý do enhance cần tracing chi tiết theo từng bước.

```mermaid
sequenceDiagram
    actor Ops as Đội vận hành
    participant CartLog as Log cart-service
    participant InvLog as Log inventory-service
    participant PayLog as Log payment-service

    Ops->>CartLog: Xem log giờ cao điểm, thấy nhiều order được tạo
    CartLog-->>Ops: Không thấy tổng thời gian mỗi order xử lý hết bao lâu
    Ops->>InvLog: Xem log inventory-service, đoán có phải do tranh chấp tồn kho
    InvLog-->>Ops: Chỉ thấy "giữ tồn kho thành công", không rõ đã chờ lock bao lâu
    Ops->>PayLog: Xem log payment-service, đoán có phải do cổng thanh toán chậm
    PayLog-->>Ops: Chỉ thấy thời điểm gọi và kết quả, không tách được độ trễ của gateway và của hệ thống
    Note over Ops: Không thể kết luận chính xác nút thắt nằm ở bước nào, phải đoán mò giữa nhiều nguyên nhân khả dĩ
```

---

# ERD - Enhance (distributed tracing xuyên checkout, tách nguyên nhân nghẽn)

Đây là trạng thái **enhance**: giữ nguyên `ORDER`, `INVENTORY_HOLD`, `PAYMENT_TRANSACTION` từ base, thay `SERVICE_LOG` rời rạc bằng mô hình tracing có cấu trúc:
- `TRACE`: mỗi order gắn 1 trace xuyên suốt 4 bước dù có bước chờ callback bất đồng bộ - đáp ứng yêu cầu 1.
- `SPAN`: mỗi bước là 1 span, tách riêng `wait_time_ms` (thời gian chờ lock/hàng đợi) và `processing_time_ms` (thời gian xử lý logic thật) để phân biệt "chậm vì tranh chấp" với "chậm vì code nặng" - đáp ứng yêu cầu 2; có `is_external_dependency` và `external_sla_ms` để đánh dấu bước gọi cổng thanh toán là phụ thuộc ngoài với SLA riêng - đáp ứng yêu cầu 3.
- `STEP_LATENCY_ROLLUP`: bảng tổng hợp độ trễ p50/p95/p99 theo từng bước theo khung thời gian, phục vụ dashboard toàn traffic (không theo từng order riêng lẻ) - đáp ứng yêu cầu 4.
- `PAYMENT_CALLBACK_EVENT`: ghi lại callback bất đồng bộ từ cổng thanh toán, liên kết ngược về span thanh toán qua `trace_id`/`span_id`, để đối chiếu khi order bị timeout nhưng cổng thanh toán thực ra đã xử lý thành công - đáp ứng yêu cầu 5.

```mermaid
erDiagram
    ORDER ||--o| TRACE : "có 1 trace theo dõi hành trình"
    TRACE ||--o{ SPAN : "gồm nhiều span theo từng bước"
    SPAN ||--o{ SPAN : "parent_span_id tự tham chiếu"
    SPAN ||--o{ PAYMENT_CALLBACK_EVENT : "đối chiếu callback trễ (nếu là span thanh toán)"
    SPAN }o--|| STEP_LATENCY_ROLLUP : "tổng hợp vào rollup theo bước/thời gian"

    ORDER {
        string order_id PK
        string cart_id
        string status
        datetime created_at
    }
    TRACE {
        string trace_id PK
        string order_id FK
        datetime started_at
    }
    SPAN {
        string span_id PK
        string trace_id FK
        string parent_span_id FK
        string service_name
        string operation_name
        datetime start_time
        datetime end_time
        int wait_time_ms
        int processing_time_ms
        boolean is_external_dependency
        int external_sla_ms
        string status
    }
    STEP_LATENCY_ROLLUP {
        string rollup_id PK
        string step_name
        datetime time_bucket
        int p50_ms
        int p95_ms
        int p99_ms
        int sample_count
    }
    PAYMENT_CALLBACK_EVENT {
        string callback_id PK
        string order_id FK
        string trace_id FK
        string span_id FK
        string gateway_reference
        datetime received_at
        string result
        boolean matched_after_timeout
    }
```

---

# Sequence Diagram - Enhance: checkout-order

Đây là flow **enhance** của `checkout-order` (so với base ở file `2-base-checkout-order-sqd.md`): `trace_id` được giữ nguyên xuyên suốt cả 4 bước, kể cả đoạn `payment-service` phải chờ callback bất đồng bộ từ cổng thanh toán thay vì trả lời đồng bộ ngay, giúp dựng lại được toàn bộ hành trình của 1 đơn hàng cụ thể khi khách hàng báo lỗi. Đáp ứng yêu cầu 1.

```mermaid
sequenceDiagram
    actor Customer
    participant Cart as cart-service
    participant Inv as inventory-service
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài
    participant Confirm as order-confirmation-service

    Customer->>Cart: Checkout giỏ hàng
    Cart->>Cart: Sinh trace_id=TR9, tạo span S1
    Cart->>Inv: Yêu cầu giữ tồn kho, header trace_id=TR9, parent_span_id=S1
    Inv->>Inv: Tạo span S2 (trace_id=TR9, parent_span_id=S1)
    Inv-->>Cart: Giữ tồn kho thành công, kết thúc span S2
    Cart->>Pay: Yêu cầu thanh toán, header trace_id=TR9, parent_span_id=S1
    Pay->>Pay: Tạo span S3 (trace_id=TR9, parent_span_id=S1)
    Pay->>Gateway: Gọi cổng thanh toán, kèm trace_id=TR9 trong metadata request
    Note over Pay,Gateway: payment-service không chờ đồng bộ, span S3 tạm ở trạng thái pending callback
    Gateway-->>Pay: Callback bất đồng bộ báo thanh toán thành công, kèm trace_id=TR9
    Pay->>Pay: Kết thúc span S3 khi nhận callback
    Pay-->>Cart: Thanh toán OK
    Cart->>Confirm: Yêu cầu xác nhận đơn hàng, header trace_id=TR9, parent_span_id=S1
    Confirm->>Confirm: Tạo span S4 (trace_id=TR9, parent_span_id=S1)
    Confirm-->>Customer: Đơn hàng đã xác nhận, kết thúc span S4 và trace TR9
```

---

# Sequence Diagram - Enhance: inventory-lock-wait-vs-processing

Đây là flow **enhance mới**: hai order cùng tranh chấp giữ tồn kho cho cùng 1 sản phẩm hot, span của `inventory-service` tách rõ `wait_time_ms` (thời gian order 2 phải chờ lock được order 1 nhả) khỏi `processing_time_ms` (thời gian xử lý logic giữ tồn kho thật sự), giúp phân biệt "chậm vì tranh chấp tồn kho" với "chậm vì code xử lý nặng". Đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    participant Cart1 as cart-service (order 1)
    participant Cart2 as cart-service (order 2)
    participant Inv as inventory-service
    participant Lock as Lock sản phẩm hot (SKU-X)

    Cart1->>Inv: Yêu cầu giữ tồn kho SKU-X, trace_id=TR1
    Inv->>Lock: Acquire lock SKU-X
    Lock-->>Inv: Cấp lock ngay (không ai giữ)
    Inv->>Inv: Xử lý giữ tồn kho thật (processing_time_ms=20ms)
    Cart2->>Inv: Yêu cầu giữ tồn kho SKU-X, trace_id=TR2 (gần như cùng lúc)
    Inv->>Lock: Acquire lock SKU-X cho order 2
    Lock-->>Inv: Phải chờ, order 1 đang giữ lock
    Inv->>Inv: Span order 2 bắt đầu đếm wait_time_ms
    Inv-->>Cart1: Giữ tồn kho order 1 thành công, release lock
    Lock-->>Inv: Cấp lock cho order 2 (đã chờ wait_time_ms=180ms)
    Inv->>Inv: Kết thúc đếm chờ, bắt đầu processing_time_ms=20ms cho order 2
    Inv-->>Cart2: Giữ tồn kho order 2 thành công
    Note over Inv: Span order 2 ghi wait_time_ms=180ms, processing_time_ms=20ms riêng biệt, không gộp chung vào 1 con số
```

---

# Sequence Diagram - Enhance: payment-external-dependency

Đây là flow **enhance mới**: span của bước gọi cổng thanh toán được đánh dấu `is_external_dependency=true` kèm `external_sla_ms` riêng, để khi tổng hợp thống kê "checkout chậm ở bước nào" vào giờ cao điểm, đội vận hành phân biệt được độ trễ do cổng thanh toán đối tác (nằm ngoài tầm kiểm soát, so với SLA riêng của họ) và độ trễ do chính hệ thống nội bộ gây ra. Đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài (SLA cam kết 500ms)
    participant Tracer as Tracing SDK

    Pay->>Tracer: Bắt đầu span S3, gắn tag is_external_dependency=true, external_sla_ms=500
    Pay->>Gateway: Gọi xử lý thanh toán
    Note over Gateway: Cổng thanh toán xử lý mất 1200ms, vượt SLA cam kết 500ms
    Gateway-->>Pay: Trả kết quả thanh toán thành công
    Pay->>Tracer: Kết thúc span S3, duration=1200ms
    Tracer->>Tracer: Ghi nhận span external, so sánh duration với external_sla_ms
    Note over Tracer: Khi tổng hợp giờ cao điểm, độ trễ vượt SLA của span external này được quy đúng cho cổng thanh toán, không tính là lỗi hệ thống nội bộ
```

---

# Sequence Diagram - Enhance: payment-timeout-late-callback-reconciliation

Đây là flow **enhance mới**: order bị timeout ở bước chờ callback thanh toán và tạm thời được đánh dấu thất bại, nhưng cổng thanh toán thực ra đã xử lý thành công phía họ và gửi callback trễ sau đó. Nhờ span thanh toán và `PAYMENT_CALLBACK_EVENT` đều lưu đủ `trace_id`/`span_id`, hệ thống đối chiếu được callback trễ với đúng order để sửa lại trạng thái, tránh báo "thất bại" trong khi tiền đã bị trừ. Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    participant Cart as cart-service
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài
    participant Confirm as order-confirmation-service
    actor Customer

    Cart->>Pay: Yêu cầu thanh toán, trace_id=TR7, tạo span S3
    Pay->>Gateway: Gọi xử lý thanh toán
    Note over Pay: Chờ callback quá timeout quy định (ví dụ 10s) mà chưa nhận được phản hồi
    Pay->>Pay: Đánh dấu span S3 status=TIMEOUT, order tạm thời status=FAILED
    Pay-->>Cart: Báo thanh toán thất bại (tạm thời)
    Cart-->>Customer: Thông báo đơn hàng thất bại
    Note over Gateway: Thực ra Gateway đã xử lý thành công ngay sau đó, chỉ gửi callback bị trễ
    Gateway-->>Pay: Callback trễ báo thanh toán thành công, kèm trace_id=TR7, gateway_reference=GW-889
    Pay->>Pay: Ghi PAYMENT_CALLBACK_EVENT, liên kết ngược về span S3 qua trace_id=TR7
    Pay->>Pay: Đối chiếu, phát hiện callback trễ khớp với order đã báo FAILED, đánh dấu matched_after_timeout=true
    Pay->>Confirm: Sửa lại trạng thái order thành đã thanh toán thành công
    Confirm-->>Customer: Cập nhật lại đơn hàng đã thanh toán thành công, không cần khiếu nại
```

---

# Sequence Diagram - Enhance: realtime-latency-dashboard

Đây là flow **enhance mới**: một pipeline tổng hợp liên tục gom span từ toàn bộ order (không chỉ 1 order riêng lẻ) thành `STEP_LATENCY_ROLLUP` theo từng bước theo khung thời gian, để dashboard trả lời được câu hỏi "giờ cao điểm hôm nay chậm hơn hôm qua, chậm chủ yếu ở bước nào" bằng cách so sánh p50/p95/p99 theo thời gian thực. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    participant Spans as Luồng span từ mọi order (cart/inventory/payment/confirm)
    participant Aggregator as Rollup Aggregator
    participant Store as Step Latency Rollup Store
    participant Dash as Dashboard vận hành
    actor Ops as Đội vận hành

    loop Mỗi khung thời gian (ví dụ mỗi phút)
        Spans->>Aggregator: Đẩy toàn bộ span kết thúc trong khung thời gian này
        Aggregator->>Aggregator: Gom span theo step_name (cart/inventory/payment/confirm)
        Aggregator->>Aggregator: Tính p50, p95, p99 duration cho từng step
        Aggregator->>Store: Ghi STEP_LATENCY_ROLLUP theo step_name, time_bucket
    end
    Ops->>Dash: Mở dashboard, so sánh giờ cao điểm hôm nay với hôm qua
    Dash->>Store: Truy vấn rollup theo từng step, 2 khoảng thời gian
    Store-->>Dash: Trả về p50/p95/p99 từng step theo thời gian thực
    Dash-->>Ops: Hiển thị bước inventory-service có p95 tăng gấp đôi so với hôm qua, xác định đúng nút thắt
```
