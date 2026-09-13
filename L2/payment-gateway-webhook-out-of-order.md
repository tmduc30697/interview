# Thanh toán qua cổng bên thứ ba với webhook callback không đảm bảo thứ tự

**Hệ thống:** Một sàn e-commerce tích hợp cổng thanh toán bên thứ ba (kiểu Stripe/VNPay), nhận callback (webhook) báo kết quả giao dịch sau khi khách thanh toán.

**Vai trò của flow:** Flow xử lý callback phải cập nhật đúng trạng thái đơn hàng dựa trên webhook nhận được, kể cả khi webhook bị gửi trùng nhiều lần hoặc đến không đúng thứ tự thời gian thực tế xảy ra.

**Yêu cầu cụ thể:**
- Webhook phải được xử lý idempotent theo `transaction_id` của cổng thanh toán: nếu nhận cùng 1 webhook 3 lần (do cổng thanh toán retry vì response chậm), chỉ áp dụng thay đổi trạng thái đúng 1 lần, 2 lần sau chỉ trả 200 OK mà không làm gì thêm.
- Mô tả cụ thể tình huống thứ tự đảo lộn: webhook "payment_success" đến sau webhook "payment_refunded" của cùng giao dịch (do độ trễ mạng khác nhau) — yêu cầu dùng timestamp hoặc sequence number từ phía cổng thanh toán để xác định thứ tự thật, không dùng thời điểm hệ thống nhận được webhook để quyết định trạng thái cuối cùng.
- Khi webhook báo thành công đến gần như đồng thời với việc khách bấm "Hủy đơn" ở app, quy định rõ ai thắng: nếu tiền đã thực sự được trừ ở cổng thanh toán (webhook success là nguồn sự thật), đơn hàng không được hủy mà phải chuyển sang luồng hoàn tiền, không đơn giản đảo trạng thái theo request đến sau.
- Nếu server nội bộ bị down đúng lúc cổng thanh toán gửi webhook, phải đảm bảo cổng thanh toán sẽ retry theo cơ chế của họ (hoặc tự chủ động polling trạng thái giao dịch định kỳ như phương án dự phòng) — không để đơn hàng bị "treo" vĩnh viễn ở trạng thái "đang xử lý" chỉ vì lỡ mất đúng 1 webhook.
- Có job đối soát hàng ngày so sánh danh sách giao dịch thành công ghi nhận nội bộ với báo cáo từ cổng thanh toán, phát hiện và cảnh báo các giao dịch lệch (có ở 1 bên nhưng không có ở bên kia).

---

# Base ERD — E-commerce tích hợp cổng thanh toán, xử lý webhook đơn giản

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có xử lý webhook idempotent/theo đúng thứ tự thật. Đề bài nói tới sàn e-commerce nhận callback từ cổng thanh toán bên thứ ba báo kết quả giao dịch — nên base cần đủ: khách hàng, đơn hàng, và giao dịch thanh toán gắn với đơn hàng. Base **chưa có** cơ chế lưu vết từng webhook nhận được để chống trùng/xác định thứ tự, chưa có job đối soát, chưa có polling dự phòng — những thứ đó là phần enhance.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ PAYMENT_TRANSACTION : "paid via"

    CUSTOMER {
        string id PK
        string name
    }
    ORDER {
        string id PK
        string customer_id FK
        decimal amount
        string status "pending | processing | paid | cancelled | refunded"
    }
    PAYMENT_TRANSACTION {
        string id PK
        string order_id FK
        string gateway_transaction_id
        string status "pending | success | refunded | failed"
        datetime updated_at
    }
```

---

# Base sequence — Xử lý webhook đơn giản (không idempotent, không xét thứ tự thật)

Đây là **base**, flow "Nhận webhook từ cổng thanh toán" ở trạng thái hiện tại — app cập nhật thẳng `ORDER.status`/`PAYMENT_TRANSACTION.status` theo nội dung webhook vừa nhận, không kiểm tra đã xử lý `transaction_id` này chưa, không xét thứ tự thật của sự kiện phía cổng thanh toán mà chỉ dựa vào thứ tự app nhận được. Flow này liên quan mật thiết tới enhance vì toàn bộ 2 yêu cầu đầu (idempotent theo transaction_id, xác định đúng thứ tự thật) đều nhằm sửa lỗ hổng ở đây.

```mermaid
sequenceDiagram
    participant Gateway as Cổng thanh toán
    participant App as Order Service
    participant DB as ORDER + PAYMENT_TRANSACTION store

    Gateway->>App: Webhook payment_success (transaction_id=TX-1)
    App->>DB: UPDATE PAYMENT_TRANSACTION SET status=success, ORDER SET status=paid
    App-->>Gateway: 200 OK

    Note over Gateway,App: Cổng thanh toán không nhận được response kịp thời, coi như thất bại và retry

    Gateway->>App: Webhook payment_success (transaction_id=TX-1) - lần retry 1
    App->>DB: UPDATE PAYMENT_TRANSACTION SET status=success, ORDER SET status=paid (chạy lại toàn bộ side-effect, vd cộng điểm thưởng, gửi email xác nhận lần 2)
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-1) - lần retry 2
    App->>DB: UPDATE lại lần 3, side-effect chạy lại lần 3
    App-->>Gateway: 200 OK

    Note over Gateway,App: Sau đó, webhook payment_refunded (xảy ra thật lúc T2) đến trước, rồi webhook payment_success cũ hơn (xảy ra thật lúc T1 < T2) đến sau do độ trễ mạng khác biệt

    Gateway->>App: Webhook payment_refunded (transaction_id=TX-2)
    App->>DB: UPDATE ORDER SET status=refunded
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-2, xảy ra thật trước refunded nhưng đến sau)
    App->>DB: UPDATE ORDER SET status=paid (ghi đè theo thời điểm nhận, không theo thời điểm xảy ra thật)
    App-->>Gateway: 200 OK

    Note over DB: Đơn hàng kết thúc ở trạng thái "paid" dù thực tế đã được hoàn tiền — sai vì dùng thời điểm nhận webhook thay vì thứ tự thật từ cổng thanh toán
```

---

# Base sequence — Hủy đơn hàng (last-write-wins, không xét nguồn sự thật)

Đây là **base**, flow "Khách bấm Hủy đơn" ở trạng thái hiện tại — app cập nhật thẳng `ORDER.status = cancelled` theo request của khách, không kiểm tra webhook thanh toán có đang/đã xử lý song song hay không. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 3 của đề bài quy định rõ ai thắng khi request hủy đơn chạm webhook success gần như đồng thời — mà base hiện chưa có quy tắc nào cả, ai ghi sau thắng.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Order Service
    participant DB as ORDER store
    participant Gateway as Cổng thanh toán

    par Khách bấm hủy đơn
        Customer->>App: Hủy đơn hàng #123
        App->>DB: UPDATE ORDER SET status=cancelled
    and Gần như đồng thời, cổng thanh toán báo đã trừ tiền thành công
        Gateway->>App: Webhook payment_success (order #123)
        App->>DB: UPDATE ORDER SET status=paid
    end

    Note over DB: Ai ghi sau thắng — nếu update "cancelled" chạy sau, đơn hàng hiển thị đã hủy dù tiền thực tế đã bị trừ ở cổng thanh toán, khách mất tiền mà tưởng đơn đã hủy
```

---

# Enhance ERD — Thêm log webhook idempotent theo thứ tự thật và đối soát hàng ngày

Đây là **enhance**, mô hình dữ liệu sau khi áp cơ chế xử lý webhook idempotent + đúng thứ tự thật lên base. So với base, thêm mới `WEBHOOK_EVENT` — lưu lại từng webhook nhận được kèm `gateway_event_id` unique (chống xử lý trùng) và `gateway_sequence_number`/`gateway_event_at` (thứ tự thật từ phía cổng thanh toán, dùng để quyết định trạng thái cuối thay vì thời điểm app nhận). `PAYMENT_TRANSACTION` thêm `last_applied_sequence_number` để biết event nào đã áp dụng gần nhất, tránh bị 1 event cũ hơn đến muộn ghi đè ngược. `ORDER` thêm trạng thái `refund_pending` cho luồng hoàn tiền khi hủy đơn đụng webhook success. Thêm mới `RECONCILIATION_DISCREPANCY` cho job đối soát hàng ngày.

```mermaid
erDiagram
    CUSTOMER ||--o{ ORDER : places
    ORDER ||--o{ PAYMENT_TRANSACTION : "paid via"
    ORDER ||--o{ WEBHOOK_EVENT : "receives"
    PAYMENT_TRANSACTION ||--o{ WEBHOOK_EVENT : "derived from"

    CUSTOMER {
        string id PK
        string name
    }
    ORDER {
        string id PK
        string customer_id FK
        decimal amount
        string status "pending | processing | paid | cancelled | refund_pending | refunded"
    }
    PAYMENT_TRANSACTION {
        string id PK
        string order_id FK
        string gateway_transaction_id
        string status "pending | success | refunded | failed"
        int last_applied_sequence_number "sequence number của webhook mới nhất đã áp dụng"
        datetime updated_at
    }
    WEBHOOK_EVENT {
        string id PK
        string order_id FK
        string payment_transaction_id FK
        string gateway_event_id UK "idempotency key, chống xử lý trùng"
        string event_type "payment_success | payment_refunded | payment_failed"
        int gateway_sequence_number "thứ tự thật do cổng thanh toán phát ra"
        datetime gateway_event_at "timestamp thật của sự kiện tại cổng thanh toán"
        datetime received_at "thời điểm app nhận được, chỉ để log, không dùng để quyết định trạng thái"
        boolean applied "đã áp dụng vào PAYMENT_TRANSACTION hay bị bỏ qua vì đến muộn hơn event đã áp dụng"
    }
    RECONCILIATION_DISCREPANCY {
        string id PK
        string gateway_transaction_id
        string order_id FK "nullable, có thể null nếu giao dịch chỉ tồn tại ở phía cổng thanh toán"
        string discrepancy_type "missing_internally | missing_at_gateway"
        date report_date
        string status "open | resolved"
        datetime detected_at
    }
```

---

# Enhance sequence — Webhook idempotent + xác định thứ tự theo sequence number thật

Đây là **enhance**, flow "Nhận webhook từ cổng thanh toán" sau khi áp idempotency + ordering theo nguồn sự thật. So với base, flow này thay đổi ở 2 điểm: (1) kiểm tra `gateway_event_id` đã tồn tại trong `WEBHOOK_EVENT` chưa trước khi áp dụng thay đổi, webhook trùng chỉ trả 200 OK mà không làm gì thêm, (2) chỉ áp dụng thay đổi nếu `gateway_sequence_number` của webhook mới lớn hơn `last_applied_sequence_number` hiện tại — quyết định theo thứ tự thật phía cổng thanh toán, không theo thời điểm app nhận.

```mermaid
sequenceDiagram
    participant Gateway as Cổng thanh toán
    participant App as Order Service
    participant DB as WEBHOOK_EVENT + PAYMENT_TRANSACTION store

    Gateway->>App: Webhook payment_success (transaction_id=TX-1, gateway_event_id=EVT-1)
    App->>DB: Kiểm tra gateway_event_id=EVT-1 đã tồn tại chưa

    DB-->>App: Chưa tồn tại
    App->>DB: Ghi WEBHOOK_EVENT(EVT-1, applied=true), UPDATE PAYMENT_TRANSACTION status=success, last_applied_sequence_number=1
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-1, gateway_event_id=EVT-1) - retry do response chậm
    App->>DB: Kiểm tra gateway_event_id=EVT-1 đã tồn tại chưa
    DB-->>App: Đã tồn tại, applied=true
    App-->>Gateway: 200 OK (idempotent, không chạy lại side-effect)

    Note over Gateway,App: Trường hợp đảo thứ tự - webhook refunded (seq=2, xảy ra thật lúc T2) đến trước, webhook success cũ (seq=1, xảy ra thật lúc T1 < T2) đến sau do độ trễ mạng

    Gateway->>App: Webhook payment_refunded (transaction_id=TX-2, gateway_event_id=EVT-2, sequence=2)
    App->>DB: EVT-2 chưa tồn tại, sequence=2 > last_applied_sequence_number hiện tại (0)
    App->>DB: Ghi WEBHOOK_EVENT(EVT-2, applied=true), UPDATE PAYMENT_TRANSACTION status=refunded, last_applied_sequence_number=2
    App-->>Gateway: 200 OK

    Gateway->>App: Webhook payment_success (transaction_id=TX-2, gateway_event_id=EVT-0, sequence=1, đến muộn)
    App->>DB: EVT-0 chưa tồn tại, nhưng sequence=1 <= last_applied_sequence_number hiện tại (2)
    App->>DB: Ghi WEBHOOK_EVENT(EVT-0, applied=false, lý do "đến muộn hơn event đã áp dụng"), không đổi PAYMENT_TRANSACTION
    App-->>Gateway: 200 OK

    Note over DB: Đơn hàng giữ đúng trạng thái refunded dù webhook success đến sau cùng, vì quyết định dựa trên sequence number thật chứ không theo thời điểm nhận
```

---

# Enhance sequence — Hủy đơn: webhook success là nguồn sự thật, chuyển sang hoàn tiền

Đây là **enhance**, flow "Khách bấm Hủy đơn" sau khi có quy tắc phân định rõ ai thắng khi đụng webhook thanh toán. So với base (last-write-wins), flow này thay đổi ở chỗ: trước khi hủy, app kiểm tra `PAYMENT_TRANSACTION.status` — nếu webhook success đã áp dụng (tiền đã thực sự bị trừ ở cổng thanh toán, nguồn sự thật), đơn hàng **không được hủy trực tiếp** mà chuyển sang `refund_pending` và khởi tạo luồng hoàn tiền, thay vì đảo trạng thái đơn giản theo request đến sau.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Order Service
    participant DB as ORDER + PAYMENT_TRANSACTION store
    participant Gateway as Cổng thanh toán

    par Khách bấm hủy đơn
        Customer->>App: Hủy đơn hàng #123
        App->>DB: Đọc PAYMENT_TRANSACTION.status hiện tại của đơn #123
    and Gần như đồng thời, webhook payment_success được áp dụng (theo flow ở file 5-enhance-webhook-callback-sqd.md)
        Gateway->>App: Webhook payment_success (order #123)
        App->>DB: UPDATE PAYMENT_TRANSACTION status=success, ORDER status=paid
    end

    alt PAYMENT_TRANSACTION.status vẫn là pending tại thời điểm đọc (chưa có webhook success nào áp dụng)
        App->>DB: UPDATE ORDER SET status=cancelled
        App-->>Customer: "Đơn hàng đã hủy"
    else PAYMENT_TRANSACTION.status đã là success (tiền đã bị trừ thật ở cổng thanh toán)
        App->>DB: UPDATE ORDER SET status=refund_pending
        App->>Gateway: Khởi tạo yêu cầu hoàn tiền
        App-->>Customer: "Đơn đã thanh toán, đang xử lý hoàn tiền thay vì hủy trực tiếp"
    end

    Note over App,DB: Webhook success luôn là nguồn sự thật — không đảo trạng thái đơn hàng chỉ vì request hủy ghi xuống sau
```

---

# Enhance sequence — Server down khi nhận webhook, retry của cổng + polling dự phòng

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — xử lý tình huống server nội bộ down đúng lúc cổng thanh toán gửi webhook, đảm bảo đơn hàng không bị "treo" vĩnh viễn ở trạng thái `processing` chỉ vì lỡ mất đúng 1 webhook. Dựa vào 2 lớp phòng vệ: cơ chế retry sẵn có của cổng thanh toán, và job polling chủ động của app cho các đơn hàng bị kẹt lâu.

```mermaid
sequenceDiagram
    participant Gateway as Cổng thanh toán
    participant App as Order Service
    participant DB as ORDER + PAYMENT_TRANSACTION store
    participant Poller as Status Polling Job (định kỳ)

    Gateway->>App: Webhook payment_success (order #456)
    Note over App: Server nội bộ đang down, request không đến được / timeout
    Gateway->>Gateway: Không nhận được 200 OK, lên lịch retry theo cơ chế riêng của cổng thanh toán

    loop Cổng thanh toán tự retry theo backoff riêng của họ
        Gateway->>App: Webhook payment_success (order #456) - retry
    end

    alt Server hồi phục trước khi cổng thanh toán retry lần kế tiếp
        Gateway->>App: Webhook payment_success (order #456) - retry thành công
        App->>DB: Xử lý idempotent như flow 5-enhance-webhook-callback-sqd.md, UPDATE status=success
        App-->>Gateway: 200 OK
    else Cổng thanh toán hết lượt retry hoặc độ trễ quá lâu trước khi server hồi phục
        Note over Poller,DB: Job polling định kỳ quét các ORDER đang ở status=processing quá 1 khoảng thời gian ngưỡng
        Poller->>DB: Tìm ORDER #456 đang processing quá lâu
        Poller->>Gateway: Chủ động gọi API truy vấn trạng thái giao dịch theo gateway_transaction_id
        Gateway-->>Poller: Trạng thái thật = success
        Poller->>DB: Áp dụng cập nhật tương tự webhook (idempotent theo gateway_transaction_id), UPDATE status=success
    end

    Note over Poller,DB: Nhờ polling dự phòng, đơn hàng không bao giờ treo vĩnh viễn chỉ vì lỡ đúng 1 webhook lúc server down
```

---

# Enhance sequence — Job đối soát hàng ngày với báo cáo cổng thanh toán

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — job chạy hàng ngày so sánh danh sách giao dịch thành công ghi nhận nội bộ (`PAYMENT_TRANSACTION`) với báo cáo giao dịch thành công từ cổng thanh toán, phát hiện và cảnh báo các giao dịch chỉ có ở 1 bên (lệch do webhook bị mất hoàn toàn kể cả sau khi có polling dự phòng, hoặc do lỗi khác).

```mermaid
sequenceDiagram
    participant Scheduler as Reconciliation Job (hàng ngày)
    participant Gateway as Cổng thanh toán
    participant DB as PAYMENT_TRANSACTION store
    participant Discrepancy as RECONCILIATION_DISCREPANCY store
    participant Ops as Đội vận hành

    Scheduler->>Gateway: Tải báo cáo giao dịch thành công trong ngày (theo gateway_transaction_id)
    Gateway-->>Scheduler: Danh sách giao dịch thành công phía cổng thanh toán
    Scheduler->>DB: Đọc danh sách PAYMENT_TRANSACTION status=success ghi nhận nội bộ trong ngày

    Scheduler->>Scheduler: So sánh 2 danh sách theo gateway_transaction_id

    alt Có giao dịch chỉ tồn tại ở báo cáo cổng thanh toán, không có nội bộ
        Scheduler->>Discrepancy: Ghi RECONCILIATION_DISCREPANCY(type=missing_internally)
    else Có giao dịch chỉ tồn tại nội bộ, không có ở báo cáo cổng thanh toán
        Scheduler->>Discrepancy: Ghi RECONCILIATION_DISCREPANCY(type=missing_at_gateway)
    end

    Discrepancy-->>Ops: Gửi cảnh báo danh sách giao dịch lệch trong ngày
    Ops->>Ops: Điều tra nguyên nhân lệch (webhook mất hẳn, lỗi ghi nhận, gian lận...)

    Note over Scheduler,DB: Job chỉ phát hiện và cảnh báo, không tự động sửa trạng thái giao dịch, tránh che giấu nguyên nhân gốc
```
