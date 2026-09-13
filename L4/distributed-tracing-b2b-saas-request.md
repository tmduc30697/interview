# Truy vết request xuyên nhiều service trong nền tảng SaaS B2B

**Hệ thống:** Một nền tảng SaaS quản lý dự án, một request tạo task đi qua service `api-gateway` → `task-service` → `notification-service` → `search-indexer` (bất đồng bộ).

**Vai trò của flow:** Distributed tracing dùng để nối các bước xử lý rải rác trên nhiều service thành một trace duy nhất, giúp debug khi một request "tạo task" bị chậm hoặc lỗi mà không rõ nghẽn ở đâu.

**Yêu cầu cụ thể:**
- Mỗi request phải được gắn một `trace_id` sinh ở gateway (nếu chưa có từ client) và propagate qua toàn bộ header của các lời gọi nội bộ tiếp theo, bao gồm cả lời gọi bất đồng bộ qua message queue.
- Mỗi service khi xử lý phải tạo một `span` con gắn với `trace_id` gốc và `parent_span_id` đúng, để dựng lại được cây gọi (call tree) đúng thứ tự và đúng quan hệ cha-con, kể cả khi các service gọi song song nhau.
- Xử lý được trường hợp một service không hỗ trợ tracing (legacy service bên thứ ba) — vẫn phải giữ được `trace_id` liên tục qua service đó (pass-through header) dù không có span chi tiết bên trong.
- Đảm bảo dữ liệu trace không làm rò rỉ thông tin nhạy cảm (không được gắn giá trị field chứa password/token vào tên span hay tag) — có allowlist rõ ràng cho các attribute được phép ghi vào trace.
- Tracing pipeline phải chịu được việc một service gửi trace data trễ hoặc mất gói (network drop) mà không làm sai lệch cây trace của các request khác — mỗi span phải tự đứng độc lập được, không phụ thuộc thứ tự nhận.

---

# ERD - Base (trước khi có distributed tracing)

Đây là trạng thái **base**: nền tảng SaaS quản lý dự án đã có luồng tạo `TASK` đi qua nhiều service (`api-gateway`, `task-service`, `notification-service`, `search-indexer`), mỗi service tự ghi log cục bộ khi xử lý, nhưng **không có bất kỳ định danh chung nào** nối các log của cùng 1 request lại với nhau. Đây là tiền đề khiến enhance cần thêm `trace_id`/`span` để dựng lại được toàn bộ hành trình của 1 request.

```mermaid
erDiagram
    TASK ||--o{ SERVICE_LOG : "phát sinh log rải rác khi xử lý"

    TASK {
        string task_id PK
        string project_id
        string title
        string status
        datetime created_at
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

# Sequence Diagram - Base: create-task

Đây là flow **base**: request tạo task đi qua `api-gateway` -> `task-service` -> `notification-service` -> `search-indexer` (bất đồng bộ qua message queue), mỗi service chỉ ghi log cục bộ của riêng nó, không kèm bất kỳ ID chung nào. Flow này là nền để so sánh với enhance, nơi mỗi bước sẽ được gắn `trace_id`/`span` xuyên suốt.

```mermaid
sequenceDiagram
    actor Client
    participant GW as api-gateway
    participant TS as task-service
    participant MQ as Message Queue
    participant NS as notification-service
    participant SI as search-indexer

    Client->>GW: POST /tasks (tạo task mới)
    GW->>TS: Gọi tạo task
    TS->>TS: Ghi log cục bộ "task created"
    TS->>MQ: Publish event "task.created"
    TS-->>GW: 201 Created
    GW-->>Client: 201 Created
    MQ-->>NS: Deliver event "task.created"
    NS->>NS: Ghi log cục bộ "gửi notification"
    MQ-->>SI: Deliver event "task.created"
    SI->>SI: Ghi log cục bộ "index task vào search"
```

---

# Sequence Diagram - Base: debug-slow-request

Đây là flow **base** minh hoạ chính vấn đề mà đề bài cần giải quyết: khi khách hàng báo request tạo task bị chậm, người vận hành phải tự tra log rời rạc trên từng service theo timestamp gần đúng và đoán mò, vì không có `trace_id` nào nối các log của cùng 1 request. Đây là lý do enhance cần distributed tracing.

```mermaid
sequenceDiagram
    actor Ops as Người vận hành
    participant GWLog as Log của api-gateway
    participant TSLog as Log của task-service
    participant NSLog as Log của notification-service
    participant SILog as Log của search-indexer

    Ops->>GWLog: Tìm log quanh thời điểm khách hàng báo lỗi
    GWLog-->>Ops: Thấy request POST /tasks lúc 10:00:01
    Ops->>TSLog: Tìm log task-service quanh 10:00:01, không có ID để lọc chính xác
    TSLog-->>Ops: Nhiều request khác cùng lúc, khó xác định đúng request nào
    Ops->>NSLog: Tìm log notification-service, đoán theo thời gian
    NSLog-->>Ops: Không chắc log này có phải cùng request hay không
    Ops->>SILog: Tìm log search-indexer, tiếp tục đoán
    SILog-->>Ops: Không xác định được service nào gây chậm
    Note over Ops: Không dựng lại được đường đi thật của request, khó xác định nghẽn ở đâu
```

---

# ERD - Enhance (distributed tracing xuyên các service)

Đây là trạng thái **enhance**: giữ nguyên `TASK` từ base, thay `SERVICE_LOG` rời rạc bằng mô hình tracing có cấu trúc:
- `TRACE`: đại diện cho toàn bộ hành trình 1 request, có `trace_id` sinh ở gateway (hoặc nhận từ client) - đáp ứng yêu cầu 1.
- `SPAN`: mỗi service tạo 1 span con gắn `trace_id` gốc và `parent_span_id`, có `is_synthetic_passthrough` để đánh dấu đoạn đi qua service không hỗ trợ tracing (chỉ pass-through header, không có span chi tiết) - đáp ứng yêu cầu 2 và 3. Span tự chứa đủ thông tin (`trace_id`, `span_id`, `start_time`, `end_time`, `status`) để đứng độc lập, không phụ thuộc thứ tự nhận - đáp ứng yêu cầu 5.
- `SPAN_ATTRIBUTE`: cặp key/value gắn vào span, chỉ được ghi nếu `key` nằm trong `TRACE_ATTRIBUTE_ALLOWLIST` - đáp ứng yêu cầu 4.
- `TRACE_ATTRIBUTE_ALLOWLIST`: danh sách các attribute key được phép ghi vào trace, dùng để lọc trước khi ghi `SPAN_ATTRIBUTE`, ngăn rò rỉ password/token - đáp ứng yêu cầu 4.
- `MQ_MESSAGE_HEADER`: đại diện cho việc propagate `trace_id`/`parent_span_id` qua header của message queue khi gọi bất đồng bộ - đáp ứng yêu cầu 1.

```mermaid
erDiagram
    TASK ||--o{ SPAN : "được xử lý qua các span"
    TRACE ||--o{ SPAN : "gồm nhiều span con"
    SPAN ||--o{ SPAN : "parent_span_id tự tham chiếu"
    SPAN ||--o{ SPAN_ATTRIBUTE : "gắn attribute"
    TRACE_ATTRIBUTE_ALLOWLIST ||--o{ SPAN_ATTRIBUTE : "kiểm tra trước khi ghi"
    SPAN ||--o| MQ_MESSAGE_HEADER : "propagate qua message queue"

    TASK {
        string task_id PK
        string project_id
        string title
        string status
        datetime created_at
    }
    TRACE {
        string trace_id PK
        string root_service
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
        string status
        boolean is_synthetic_passthrough
    }
    SPAN_ATTRIBUTE {
        string span_id FK
        string key
        string value
    }
    TRACE_ATTRIBUTE_ALLOWLIST {
        string key PK
        string description
    }
    MQ_MESSAGE_HEADER {
        string message_id PK
        string trace_id FK
        string parent_span_id FK
    }
```

---

# Sequence Diagram - Enhance: create-task

Đây là flow **enhance** của `create-task` (so với base ở file `2-base-create-task-sqd.md`): `api-gateway` sinh `trace_id` nếu client chưa gửi, mỗi service tạo span con gắn `parent_span_id` đúng (kể cả khi `notification-service` và `search-indexer` được gọi song song qua message queue), và `trace_id`/`parent_span_id` được propagate qua cả header HTTP lẫn header của message queue. Đáp ứng yêu cầu 1 (propagate trace_id qua header, kể cả async qua MQ) và yêu cầu 2 (mỗi service tạo span con đúng quan hệ cha-con, kể cả gọi song song).

```mermaid
sequenceDiagram
    actor Client
    participant GW as api-gateway
    participant TS as task-service
    participant MQ as Message Queue
    participant NS as notification-service
    participant SI as search-indexer

    Client->>GW: POST /tasks (không kèm trace_id)
    GW->>GW: Sinh trace_id=T1, tạo span S1 (parent=null)
    GW->>TS: Gọi tạo task, header: trace_id=T1, parent_span_id=S1
    TS->>TS: Tạo span S2 (trace_id=T1, parent_span_id=S1)
    TS->>MQ: Publish event "task.created", header: trace_id=T1, parent_span_id=S2
    TS-->>GW: 201 Created, kết thúc span S2
    GW-->>Client: 201 Created, kết thúc span S1
    par Xử lý song song từ message queue
        MQ-->>NS: Deliver event, header trace_id=T1, parent_span_id=S2
        NS->>NS: Tạo span S3 (trace_id=T1, parent_span_id=S2), gửi notification
    and
        MQ-->>SI: Deliver event, header trace_id=T1, parent_span_id=S2
        SI->>SI: Tạo span S4 (trace_id=T1, parent_span_id=S2), index task
    end
    Note over TS,SI: Cây span dựng lại được: S1 -> S2 -> (S3, S4) chạy song song
```

---

# Sequence Diagram - Enhance: legacy-service-passthrough

Đây là flow **enhance mới**: `task-service` gọi thêm một service bên thứ ba đã cũ (`legacy-billing-check`) không hỗ trợ tracing, không tạo span nào cả, nhưng vẫn nhận và trả lại nguyên vẹn header `trace_id` (pass-through) để chuỗi trace không bị đứt quãng khi các service tiếp theo tiếp tục xử lý. Đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant TS as task-service
    participant Legacy as legacy-billing-check (không hỗ trợ tracing)
    participant NS as notification-service

    TS->>TS: Đang xử lý trong span S2 (trace_id=T1, parent_span_id=S1)
    TS->>Legacy: Gọi kiểm tra billing, header: trace_id=T1, parent_span_id=S2
    Note over Legacy: Service legacy không đọc, không tạo span, chỉ xử lý nghiệp vụ
    Legacy-->>TS: Trả kết quả, header trace_id=T1 được giữ nguyên (pass-through)
    TS->>NS: Gọi tiếp notification-service, header: trace_id=T1, parent_span_id=S2
    NS->>NS: Tạo span S3 (trace_id=T1, parent_span_id=S2)
    Note over TS,NS: Chuỗi trace T1 không bị đứt dù đoạn qua legacy-billing-check không có span chi tiết
```

---

# Sequence Diagram - Enhance: sensitive-attribute-allowlist

Đây là flow **enhance mới**: khi `task-service` gắn attribute vào span (ví dụ thông tin request), tracing SDK kiểm tra từng key với `TRACE_ATTRIBUTE_ALLOWLIST` trước khi ghi - key hợp lệ như `task_id`, `project_id` được ghi bình thường, còn key nhạy cảm như `auth_token`/`password` bị chặn, không bao giờ xuất hiện trong tên span hay tag. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    participant TS as task-service
    participant SDK as Tracing SDK (local)
    participant Allow as Trace Attribute Allowlist
    participant Collector as Tracing Collector/Backend

    TS->>SDK: Gắn attribute vào span S2, key=task_id, value=T-123
    SDK->>Allow: Kiểm tra key=task_id có trong allowlist không
    Allow-->>SDK: Hợp lệ, cho phép ghi
    SDK->>Collector: Gửi span S2 kèm attribute task_id=T-123
    TS->>SDK: Gắn attribute vào span S2, key=auth_token, value=Bearer xyz...
    SDK->>Allow: Kiểm tra key=auth_token có trong allowlist không
    Allow-->>SDK: Không hợp lệ, key nhạy cảm bị chặn
    SDK->>SDK: Bỏ qua attribute auth_token, không đưa vào payload gửi đi
    Note over Collector: Collector không bao giờ nhận được giá trị auth_token/password trong span
```

---

# Sequence Diagram - Enhance: late-or-dropped-span

Đây là flow **enhance mới**: mỗi span được gửi độc lập tới collector (không phụ thuộc thứ tự nhận), nên khi span của `search-indexer` cho trace T1 bị gửi trễ do network drop, collector vẫn dựng đúng cây trace của T1 ngay khi span tới muộn, và trace T2 của 1 request khác xử lý song song hoàn toàn không bị ảnh hưởng. Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    participant SI as search-indexer (trace T1)
    participant NS as notification-service (trace T1)
    participant OtherSvc as task-service (trace T2, request khác)
    participant Collector as Tracing Collector/Backend

    NS->>Collector: Gửi span S3 (trace_id=T1, parent_span_id=S2)
    Collector->>Collector: Lưu span S3, gắn vào cây trace T1
    OtherSvc->>Collector: Gửi span của trace T2 (không liên quan T1)
    Collector->>Collector: Lưu span T2 độc lập, không ảnh hưởng tới T1
    Note over SI,Collector: Span S4 của search-indexer (trace T1) bị mất gói trên đường truyền, chưa tới collector
    Collector->>Collector: Hiển thị cây trace T1 tạm thời thiếu S4, các span khác vẫn đúng
    SI->>Collector: Gửi lại/gửi trễ span S4 (trace_id=T1, parent_span_id=S2)
    Collector->>Collector: Nhận span S4 trễ, tự bổ sung đúng vị trí vào cây trace T1 nhờ span tự chứa đủ thông tin
    Note over Collector: Trace T1 được dựng lại đầy đủ, trace T2 không bị sai lệch trong suốt quá trình
```
