# Flash sale số lượng giới hạn trên sàn e-commerce (100 sản phẩm, hàng chục nghìn người chờ)

**Hệ thống:** Một sàn e-commerce tổ chức flash sale: đúng 12:00:00 mở bán 100 sản phẩm giá sốc, dự kiến hàng chục nghìn người cùng bấm "Mua ngay" trong vài giây đầu.

**Vai trò của flow:** Flow checkout phải đảm bảo đúng 100 người mua được (không hơn không kém), xử lý đúng khi hàng chục nghìn request cùng cạnh tranh vào đúng thời điểm mở bán, không để hệ thống sập hoặc bán vượt tồn kho (overselling).

**Yêu cầu cụ thể:**
- Không cho phép dùng `SELECT tồn_kho rồi kiểm tra > 0 rồi UPDATE trừ` ở tầng application (race condition kinh điển: nhiều request đọc thấy tồn kho > 0 cùng lúc rồi đều trừ) — yêu cầu dùng update nguyên tử kiểu `UPDATE stock SET qty = qty - 1 WHERE qty > 0` và kiểm tra số dòng bị ảnh hưởng (affected rows) để biết có mua được hay không.
- Thiết kế hàng đợi (queue) hoặc rate-limiter ở tầng trước DB để giảm áp lực: chỉ cho một số lượng request giới hạn/giây chạm vào transaction trừ tồn kho, các request còn lại nhận phản hồi "đang xử lý, vui lòng chờ" thay vì để tất cả cùng đấm vào DB gây timeout hàng loạt.
- Mô tả cụ thể: 2 request đến gần như đồng thời tại thời điểm tồn kho còn đúng 1 sản phẩm — hệ thống phải đảm bảo chỉ 1 request được xác nhận mua, request kia nhận lỗi "hết hàng" trong thời gian ngắn (không phải timeout), viết test giả lập 1000 request đồng thời cho 100 sản phẩm và assert đúng 100 request thành công.
- Idempotency: nếu client retry do timeout mạng (không rõ request trước có xử lý thành công hay không), request retry với cùng idempotency key không được trừ tồn kho lần 2 — phải trả lại đúng kết quả của lần xử lý đầu.
- Có cơ chế "circuit breaker"/giảm tải khi tồn kho đã về 0: các request đến sau phải được chặn ngay từ tầng cache/API gateway, không để lọt xuống DB gây tải vô ích.

---

# ERD - Base: Checkout kiểm tra tồn kho kiểu đọc-rồi-ghi

Đây là trạng thái **base**, trước khi áp đề bài. Sàn e-commerce đã có sản phẩm với số lượng tồn kho, khách bấm "Mua ngay" thì backend đọc `stock_qty`, kiểm tra `> 0` ở tầng application rồi mới `UPDATE` trừ đi 1 — đây chính là pattern race-condition kinh điển mà đề bài yêu cầu loại bỏ. Không có hàng đợi/rate-limiter trước DB, không có idempotency key, không có circuit breaker khi hết hàng. Đây là nền tảng để đề bài (atomic update, queue/rate-limiter, idempotency, circuit breaker) có ý nghĩa khi so sánh.

```mermaid
erDiagram
    PRODUCT ||--o{ ORDER : "được đặt mua"
    USER ||--o{ ORDER : places

    PRODUCT {
        string product_id
        string name
        int stock_qty
        datetime sale_start_time
    }
    USER {
        string user_id
        string name
    }
    ORDER {
        string order_id
        string product_id
        string user_id
        string status
        datetime created_at
    }
```

Lưu ý: không có field `idempotency_key` trên `ORDER`, không có entity nào đại diện cho hàng đợi/rate-limiter trước khi chạm DB, và không có trạng thái circuit breaker khi `stock_qty` về 0 — đây chính là các khoảng trống mà enhance sẽ lấp.

---

# Sequence - Base: Mua ngay (đọc tồn kho rồi trừ, race condition)

Đây là flow **base**: hàng chục nghìn request cùng "Mua ngay" đổ thẳng xuống DB, mỗi request đọc `stock_qty`, kiểm tra `> 0` ở tầng application rồi mới `UPDATE` trừ đi 1. Khi tồn kho còn đúng 1 sản phẩm, 2 request đến gần như đồng thời đều đọc thấy `stock_qty=1 > 0`, cả 2 đều được xác nhận mua, dẫn tới overselling. Đây là tiền đề cho yêu cầu 1, 2 và 3 của đề bài.

```mermaid
sequenceDiagram
    participant U1 as Khách A
    participant U2 as Khách B
    participant API as Checkout API
    participant DB as Database

    Note over DB: stock_qty = 1 (sản phẩm cuối cùng)
    par 2 request gần như đồng thời
        U1->>API: Bấm Mua ngay
        API->>DB: SELECT stock_qty FROM PRODUCT (=1)
    and
        U2->>API: Bấm Mua ngay
        API->>DB: SELECT stock_qty FROM PRODUCT (=1, chưa thấy thay đổi)
    end
    Note over API: Cả 2 request đều thấy stock_qty=1 > 0, đều nghĩ mình mua được
    API->>DB: UPDATE stock_qty = 0 (cho khách A)
    API->>DB: Tạo ORDER cho khách A, status=confirmed
    API-->>U1: Mua thành công
    API->>DB: UPDATE stock_qty = -1 (cho khách B, ghi đè, không kiểm tra lại)
    API->>DB: Tạo ORDER cho khách B, status=confirmed
    API-->>U2: Mua thành công
    Note over DB: Overselling, 2 khách cùng mua được sản phẩm cuối cùng, stock_qty âm
```

---

# Sequence - Base: Retry sau timeout mạng (trừ tồn kho lần 2)

Đây là flow **base**: client bấm "Mua ngay", mạng chập chờn khiến response bị timeout dù request đã xử lý thành công ở backend, client không biết nên tự động retry với cùng thao tác. Vì không có idempotency key, backend coi request retry là 1 giao dịch mới hoàn toàn và trừ tồn kho lần nữa cho cùng 1 khách. Đây là tiền đề cho yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    participant U as Khách hàng
    participant API as Checkout API
    participant DB as Database

    U->>API: Bấm Mua ngay (lần 1)
    API->>DB: UPDATE stock_qty - 1, tạo ORDER #1 status=confirmed
    DB-->>API: Thành công
    API--xU: Response bị timeout trên đường mạng, khách không nhận được
    Note over U: Khách không biết request trước đã thành công hay chưa
    U->>API: Tự động retry, bấm Mua ngay lần nữa
    Note over API: Không có idempotency key để nhận diện đây là request lặp lại
    API->>DB: UPDATE stock_qty - 1 lần nữa, tạo ORDER #2 status=confirmed
    DB-->>API: Thành công
    API-->>U: Mua thành công
    Note over DB: Cùng 1 khách bị trừ tồn kho 2 lần cho 1 lần mua thực tế, tạo ra 2 ORDER trùng lặp
```

---

# ERD - Enhance: Atomic update, hàng đợi trước DB, idempotency key, circuit breaker

Đây là trạng thái **enhance**, sau khi áp toàn bộ đề bài lên base. So với base, các thay đổi chính:

- Bỏ hoàn toàn pattern đọc-rồi-ghi: `PRODUCT.stock_qty` chỉ được trừ bằng `UPDATE ... SET qty = qty - 1 WHERE qty > 0`, dựa vào affected rows để biết có mua được hay không — đáp ứng **yêu cầu 1 và 3**.
- Thêm entity `QUEUE_TICKET` — mọi request phải qua hàng đợi/rate-limiter trước khi được phép chạm vào transaction trừ tồn kho, request vượt hạn mức nhận trạng thái `waiting` thay vì đấm thẳng vào DB — đáp ứng **yêu cầu 2**.
- `ORDER` có thêm `idempotency_key` (unique theo `user_id` + key), cho phép trả lại đúng kết quả của lần xử lý đầu khi client retry — đáp ứng **yêu cầu 4**.
- Thêm entity `CIRCUIT_BREAKER_STATE` theo từng `product_id` — khi `stock_qty` về 0, breaker chuyển `open`, chặn request ngay ở tầng cache/gateway, không cho lọt xuống DB — đáp ứng **yêu cầu 5**.

```mermaid
erDiagram
    PRODUCT ||--o{ ORDER : "được đặt mua"
    PRODUCT ||--o{ QUEUE_TICKET : "xếp hàng chờ vào transaction"
    PRODUCT ||--o| CIRCUIT_BREAKER_STATE : "có trạng thái breaker riêng"
    USER ||--o{ ORDER : places
    USER ||--o{ QUEUE_TICKET : holds

    PRODUCT {
        string product_id
        string name
        int stock_qty
        datetime sale_start_time
    }
    USER {
        string user_id
        string name
    }
    ORDER {
        string order_id
        string product_id
        string user_id
        string idempotency_key
        string status
        datetime created_at
    }
    QUEUE_TICKET {
        string ticket_id
        string product_id
        string user_id
        string status
        datetime enqueued_at
        datetime allowed_at
    }
    CIRCUIT_BREAKER_STATE {
        string product_id
        string state
        datetime tripped_at
    }
```

---

# Sequence - Enhance: Mua ngay (atomic update, kiểm tra affected rows)

Đây là flow **enhance** của `buy-now-checkout` (so với base). Khác biệt so với base: không còn bước `SELECT` riêng để kiểm tra tồn kho, thay vào đó chạy thẳng `UPDATE stock_qty = stock_qty - 1 WHERE stock_qty > 0` và kiểm tra số dòng bị ảnh hưởng (affected rows) để biết có mua được hay không — DB tự đảm bảo tính atomic ở mức row-lock, nên khi tồn kho còn đúng 1, trong 2 request đến gần như đồng thời chỉ đúng 1 request có `affected_rows=1`. Đáp ứng **yêu cầu 1 và 3** của đề bài.

```mermaid
sequenceDiagram
    participant U1 as Khách A
    participant U2 as Khách B
    participant API as Checkout API
    participant DB as Database

    Note over DB: stock_qty = 1 (sản phẩm cuối cùng)
    par 2 request gần như đồng thời
        U1->>API: Bấm Mua ngay
        API->>DB: UPDATE PRODUCT SET stock_qty = stock_qty - 1 WHERE product_id=X AND stock_qty > 0
    and
        U2->>API: Bấm Mua ngay
        API->>DB: UPDATE PRODUCT SET stock_qty = stock_qty - 1 WHERE product_id=X AND stock_qty > 0
    end
    Note over DB: DB serialize 2 UPDATE trên cùng 1 row, chỉ 1 câu lệnh thấy stock_qty=1 và trừ được, câu còn lại thấy stock_qty=0 nên điều kiện WHERE không khớp
    DB-->>API: affected_rows=1 cho khách A, affected_rows=0 cho khách B
    API->>DB: Tạo ORDER cho khách A, status=confirmed
    API-->>U1: Mua thành công
    API-->>U2: Hết hàng (phản hồi ngay, không phải timeout)
    Note over DB: Đúng 1 khách mua được sản phẩm cuối, không overselling
```

---

# Sequence - Enhance: Hàng đợi/rate-limiter chặn trước khi chạm DB

Đây là flow **enhance hoàn toàn mới**, mô tả lớp hàng đợi/rate-limiter đặt trước transaction trừ tồn kho. Khi hàng chục nghìn request cùng đổ vào lúc 12:00:00, chỉ 1 số lượng giới hạn/giây được cho phép chạm vào DB, các request còn lại nhận ngay phản hồi "đang xử lý, vui lòng chờ" thay vì để tất cả cùng đấm vào DB gây timeout hàng loạt. Đáp ứng **yêu cầu 2** của đề bài.

```mermaid
sequenceDiagram
    participant Users as Hàng chục nghìn khách
    participant GW as API Gateway / Rate Limiter
    participant Queue as QUEUE_TICKET Store
    participant API as Checkout API
    participant DB as Database

    Users->>GW: Hàng chục nghìn request "Mua ngay" gần như cùng lúc
    GW->>Queue: Tạo QUEUE_TICKET(status=waiting) cho từng request
    GW-->>Users: Phản hồi ngay "Đang xử lý, vui lòng chờ" kèm ticket_id, không block request

    loop Rate limiter chỉ cho phép N request/giây chạm DB
        GW->>Queue: Lấy ticket có enqueued_at sớm nhất, status=waiting
        Queue-->>GW: ticket_id kế tiếp
        GW->>API: Cho phép ticket này chạy transaction trừ tồn kho
        API->>DB: UPDATE stock_qty - 1 WHERE stock_qty > 0
        DB-->>API: affected_rows
        API->>Queue: Cập nhật ticket status=done, kèm kết quả (mua được / hết hàng)
    end

    Users->>GW: Poll kết quả bằng ticket_id
    GW->>Queue: Tra trạng thái ticket
    Queue-->>GW: status=done, kết quả=mua được / hết hàng
    GW-->>Users: Trả kết quả cuối cùng
    Note over DB: DB chỉ nhận đúng N request/giây, không bị hàng chục nghìn kết nối đồng thời làm timeout hàng loạt
```

---

# Sequence - Enhance: Retry sau timeout mạng (idempotent, không trừ tồn kho lần 2)

Đây là flow **enhance** của `retry-after-timeout` (so với base). Khác biệt so với base: client gửi kèm `idempotency_key` cố định cho lần bấm mua đó (kể cả khi retry), backend kiểm tra key này đã tồn tại `ORDER` hay chưa trước khi chạy transaction trừ tồn kho — nếu đã tồn tại, trả thẳng lại kết quả của lần xử lý đầu tiên, không chạy lại transaction. Đáp ứng **yêu cầu 4** của đề bài.

```mermaid
sequenceDiagram
    participant U as Khách hàng
    participant API as Checkout API
    participant DB as Database

    U->>API: Bấm Mua ngay (lần 1, idempotency_key=abc123)
    API->>DB: Kiểm tra ORDER với idempotency_key=abc123 (chưa có)
    API->>DB: UPDATE stock_qty - 1 WHERE stock_qty > 0
    DB-->>API: affected_rows=1
    API->>DB: Tạo ORDER(idempotency_key=abc123, status=confirmed)
    DB-->>API: Thành công
    API--xU: Response bị timeout trên đường mạng, khách không nhận được
    Note over U: Khách không biết request trước đã thành công hay chưa
    U->>API: Tự động retry, cùng idempotency_key=abc123
    API->>DB: Kiểm tra ORDER với idempotency_key=abc123
    DB-->>API: Đã tồn tại ORDER, status=confirmed
    Note over API: Không chạy lại transaction trừ tồn kho, chỉ trả lại đúng kết quả cũ
    API-->>U: Mua thành công (kết quả của lần xử lý đầu tiên)
    Note over DB: Tồn kho chỉ bị trừ đúng 1 lần cho 1 lần mua thực tế, dù client retry bao nhiêu lần
```

---

# Sequence - Enhance: Circuit breaker chặn request khi đã hết hàng

Đây là flow **enhance hoàn toàn mới**, mô tả cơ chế circuit breaker/giảm tải khi tồn kho đã về 0. Khi request cuối cùng trừ tồn kho về 0 (hoặc thất bại vì `affected_rows=0`), hệ thống lập tức bật cờ "hết hàng" ở tầng cache/API gateway, mọi request đến sau bị chặn ngay tại đó, không để lọt xuống DB gây tải vô ích. Đáp ứng **yêu cầu 5** của đề bài.

```mermaid
sequenceDiagram
    participant API as Checkout API
    participant DB as Database
    participant Cache as Cache (CIRCUIT_BREAKER_STATE)
    participant GW as API Gateway
    participant LateUser as Khách đến sau khi đã hết hàng

    API->>DB: UPDATE stock_qty - 1 WHERE stock_qty > 0 (request cuối cùng)
    DB-->>API: affected_rows=1, stock_qty vừa về 0
    API->>Cache: Set CIRCUIT_BREAKER_STATE(product_id=X, state=open, tripped_at=now)
    Note over Cache: Breaker chuyển open ngay khi phát hiện stock_qty=0, không chờ request tiếp theo mới phát hiện

    LateUser->>GW: Bấm Mua ngay (đến sau khi đã hết hàng)
    GW->>Cache: Kiểm tra CIRCUIT_BREAKER_STATE(product_id=X)
    Cache-->>GW: state=open
    GW-->>LateUser: Hết hàng (phản hồi ngay tại gateway, không gọi xuống Checkout API hay DB)
    Note over DB: DB hoàn toàn không nhận thêm request nào cho sản phẩm X sau khi đã hết hàng, tránh tải vô ích lúc cao điểm
```
