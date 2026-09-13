# Giữ chỗ tồn kho cho sản phẩm flash sale

**Hệ thống:** Sàn thương mại điện tử mở bán flash sale một sản phẩm số lượng giới hạn, hàng nghìn request mua hàng gửi tới cùng lúc trong vài giây đầu mở bán.

**Vai trò của flow:** Distributed lock theo từng sản phẩm (per-SKU) đảm bảo tại một thời điểm chỉ một tiến trình được đọc-và-trừ tồn kho cho sản phẩm đó, tránh oversell do nhiều request đọc cùng số tồn kho rồi cùng trừ.

**Yêu cầu cụ thể:**
- Lock phải được khóa ở granularity đúng mức (per-SKU, không lock toàn catalog gây nghẽn cổ chai, không lock quá lỏng theo warehouse chung gây oversell chéo sản phẩm), với TTL đủ ngắn để nhả nhanh cho request tiếp theo nhưng đủ dài để hoàn tất thao tác trừ kho — quá ngắn thì lock hết hạn giữa chừng gây 2 tiến trình cùng trừ, quá dài thì hàng nghìn request xếp hàng chờ gây timeout hàng loạt.
- Nếu tiến trình giữ lock crash ngay sau khi đã trừ số dư tồn kho trong DB nhưng chưa kịp tạo record đơn hàng/release lock, phải có cơ chế phát hiện trạng thái dở dang này khi lock hết hạn (ví dụ đối chiếu số đã trừ với đơn hàng tồn tại tương ứng) để hoàn lại tồn kho bị "kẹt" thay vì mất vĩnh viễn.
- Dùng fencing token gắn với lock để tránh trường hợp một tiến trình bị treo (GC pause, network delay) tưởng mình còn giữ lock, ghi trừ kho sau khi lock đã hết hạn và bị tiến trình khác giành mất — thao tác trừ kho với token cũ phải bị service tồn kho từ chối.
- Với hàng nghìn request cạnh tranh cùng 1 lock, phải có chiến lược fail-fast cho request không giành được lock trong thời gian ngắn (trả "hết hàng/thử lại" thay vì xếp hàng chờ vô hạn), tránh hàng đợi lock phình to làm nghẽn toàn hệ thống.
- Retry của client (do timeout network) không được gây trừ kho 2 lần cho cùng một yêu cầu mua — cần idempotency key gắn với request mua hàng độc lập với cơ chế lock, vì lock chỉ đảm bảo tuần tự xử lý chứ không đảm bảo request không bị gửi lặp.

---

# Base ERD — Tồn kho và đơn hàng chưa có cơ chế khóa khi mua đồng thời

Đây là **base**: trạng thái sàn thương mại điện tử *trước khi* có distributed lock per-SKU. Suy luận từ đề bài, base đã có `PRODUCT` với `stock_quantity` đại diện số lượng tồn kho, và mỗi lần mua thành công tạo 1 `ORDER` tham chiếu tới product. Base xử lý mua hàng bằng đọc số tồn kho rồi trừ trực tiếp trong 1 transaction DB thông thường, không có khóa nào bảo vệ giữa lúc đọc và lúc trừ khi nhiều request cùng mua 1 sản phẩm.

```mermaid
erDiagram
    PRODUCT ||--o{ ORDER : "được mua trong"

    PRODUCT {
        string sku PK
        string name
        int stock_quantity
    }
    ORDER {
        string id PK
        string product_sku FK
        string user_id
        int quantity
        datetime created_at
        string status "confirmed|failed"
    }
```

---

# Base sequence — Đọc-trừ tồn kho trực tiếp, oversell khi mua đồng thời

Đây là **base**, flow mua hàng hiện tại: server đọc `stock_quantity`, kiểm tra còn hàng rồi trừ đi trong 1 transaction DB thông thường, không có khóa nào bảo vệ khoảng giữa đọc và ghi. Đây chính là kịch bản oversell nêu trong đề bài — hàng nghìn request mua hàng gửi tới cùng lúc khi mở bán flash sale, nhiều request cùng đọc thấy còn hàng rồi cùng trừ.

```mermaid
sequenceDiagram
    actor ReqA as Request A
    actor ReqB as Request B
    participant App as Order Service
    participant DB as Database

    Note over App,DB: Sản phẩm chỉ còn 1 đơn vị tồn kho

    ReqA->>App: Mua sản phẩm SKU-123, số lượng 1
    ReqB->>App: Mua sản phẩm SKU-123, số lượng 1 (gần như cùng lúc)

    App->>DB: (A) SELECT stock_quantity FROM PRODUCT WHERE sku=SKU-123
    DB-->>App: (A) stock_quantity = 1

    App->>DB: (B) SELECT stock_quantity FROM PRODUCT WHERE sku=SKU-123
    DB-->>App: (B) stock_quantity = 1

    Note over App: Cả 2 request đều đọc thấy còn hàng, chưa có khóa chặn giữa đọc và ghi

    App->>DB: (A) UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    App->>DB: (A) INSERT ORDER (product_sku=SKU-123, quantity=1, status=confirmed)

    App->>DB: (B) UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    App->>DB: (B) INSERT ORDER (product_sku=SKU-123, quantity=1, status=confirmed)

    DB-->>App: Cả 2 UPDATE/INSERT đều thành công
    App-->>ReqA: Mua thành công
    App-->>ReqB: Mua thành công

    Note over DB: stock_quantity bị trừ về -1, 2 đơn hàng được tạo cho 1 sản phẩm duy nhất, oversell
```

---

# Enhance ERD — Thêm idempotency, fencing token và log trừ kho để reconcile

Đây là ERD **sau khi** enhance được áp dụng lên base. `PRODUCT` giữ nguyên (khóa per-SKU trên Redis là coordination ephemeral, key dạng `lock:sku:{sku}`, không phải dữ liệu bền vững nên không xuất hiện là entity riêng). Có 2 entity mới và `ORDER` được bổ sung trường, ứng trực tiếp với các yêu cầu:

- `INVENTORY_DEDUCTION.fencing_token` và `ORDER.fencing_token_used` — đáp ứng yêu cầu 3 (fencing token gắn với lock, request dùng token cũ sau khi lock hết hạn bị từ chối ghi).
- `INVENTORY_DEDUCTION.status` (`pending`/`confirmed`/`refunded`) — đáp ứng yêu cầu 2 (khi lock hết hạn mà process crash giữa lúc đã trừ kho nhưng chưa tạo order, job reconcile phát hiện deduction còn `pending` không có order tương ứng để hoàn lại tồn kho).
- `IDEMPOTENCY_RECORD` (mới), độc lập với cơ chế lock — đáp ứng yêu cầu 5 (client retry do timeout network không được trừ kho 2 lần cho cùng 1 yêu cầu mua, dùng idempotency key gắn với request thay vì dựa vào lock).
- `PRODUCT.lock_ttl_ms` — đáp ứng yêu cầu 1 (TTL cấu hình theo từng SKU, đủ ngắn để nhả nhanh nhưng đủ dài để hoàn tất thao tác trừ kho).

```mermaid
erDiagram
    PRODUCT ||--o{ ORDER : "được mua trong"
    PRODUCT ||--o{ INVENTORY_DEDUCTION : "ghi nhận mỗi lần trừ"
    ORDER ||--o| IDEMPOTENCY_RECORD : "gắn với 1 yêu cầu mua"

    PRODUCT {
        string sku PK
        string name
        int stock_quantity
        int lock_ttl_ms "vd 300ms, riêng theo SKU"
    }
    ORDER {
        string id PK
        string product_sku FK
        string user_id
        int quantity
        int fencing_token_used
        datetime created_at
        string status "confirmed|failed"
    }
    INVENTORY_DEDUCTION {
        string id PK
        string product_sku FK
        string request_id
        int fencing_token
        int quantity_deducted
        string order_id "null cho tới khi ORDER được tạo"
        string status "pending|confirmed|refunded"
        datetime created_at
    }
    IDEMPOTENCY_RECORD {
        string idempotency_key PK
        string order_id FK
        string result_status "processing|success|failed"
        datetime created_at
        datetime expires_at
    }
```

---

# Enhance sequence — Lock per-SKU với TTL hợp lý và fail-fast cho request thua cuộc

Đây là **enhance** của flow `purchase-flash-sale` đã có ở base. So với base (đọc-trừ trực tiếp không khóa), enhance thêm distributed lock theo đúng granularity per-SKU (không lock toàn catalog, không lock chung theo warehouse) với TTL ngắn vừa đủ, và fail-fast cho request không giành được lock trong thời gian ngắn. Đáp ứng yêu cầu 1 (granularity per-SKU, TTL đủ ngắn để nhả nhanh nhưng đủ dài để hoàn tất trừ kho) và yêu cầu 4 (fail-fast, không xếp hàng chờ vô hạn khi có hàng nghìn request cạnh tranh).

```mermaid
sequenceDiagram
    actor Req1 as Request 1
    actor Req2 as Request 2
    actor ReqN as Request N (hàng nghìn request khác)
    participant App as Order Service
    participant Lock as Redis Lock (lock:sku:SKU-123, TTL 300ms)
    participant DB as Database

    Note over Req1,ReqN: Chỉ còn 1 đơn vị tồn kho SKU-123, hàng nghìn request cùng cạnh tranh

    Req1->>App: Mua SKU-123
    App->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+1
    Lock-->>App: Giành lock thành công

    par Các request khác cùng thử giành lock
        Req2->>App: Mua SKU-123
        App->>Lock: SET lock:sku:SKU-123 NX PX 300ms
        Lock-->>App: Thất bại, lock đang bị Req1 giữ
        App->>App: Chờ ngắn (vd 50ms), thử lại 1-2 lần
        App-->>Req2: Vẫn không giành được trong thời gian cho phép, trả "Hết hàng, vui lòng thử lại"

        ReqN->>App: Mua SKU-123
        App->>Lock: SET lock:sku:SKU-123 NX PX 300ms
        Lock-->>App: Thất bại
        App-->>ReqN: Fail-fast ngay, "Hết hàng, vui lòng thử lại" (không xếp hàng chờ)
    end

    App->>DB: (Req1) SELECT stock_quantity FROM PRODUCT WHERE sku=SKU-123
    DB-->>App: stock_quantity = 1
    App->>DB: UPDATE PRODUCT SET stock_quantity = 0, INSERT INVENTORY_DEDUCTION (fencing_token=N+1, status=pending)
    App->>DB: INSERT ORDER (fencing_token_used=N+1, status=confirmed)
    App->>DB: UPDATE INVENTORY_DEDUCTION SET status=confirmed, order_id=...
    App->>Lock: DEL lock:sku:SKU-123 (release ngay sau khi hoàn tất, chỉ ~vài chục ms)

    App-->>Req1: Mua thành công

    Note over App,Lock: Lock chỉ khóa đúng SKU-123, các SKU khác không bị ảnh hưởng, TTL 300ms đủ để hoàn tất trừ kho nhưng nhả rất nhanh
```

---

# Enhance sequence — Reconcile tồn kho bị kẹt khi process crash giữa chừng

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp process giữ lock crash ngay sau khi đã trừ tồn kho trong DB nhưng chưa kịp tạo `ORDER`/release lock. Đáp ứng yêu cầu 2: phát hiện trạng thái dở dang này khi lock hết hạn (đối chiếu số đã trừ với đơn hàng tồn tại tương ứng), để hoàn lại tồn kho bị kẹt thay vì mất vĩnh viễn.

```mermaid
sequenceDiagram
    actor Req as Request (process sẽ crash)
    participant App as Order Service (instance sẽ crash)
    participant Lock as Redis Lock (lock:sku:SKU-123)
    participant DB as Database
    participant Reconciler as Reconciliation Job

    Req->>App: Mua SKU-123
    App->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+1
    Lock-->>App: Giành lock thành công

    App->>DB: UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    App->>DB: INSERT INVENTORY_DEDUCTION (fencing_token=N+1, quantity_deducted=1, status=pending, order_id=null)

    Note over App: Process crash ngay tại đây, chưa kịp INSERT ORDER, chưa kịp DEL lock

    Note over Lock: Lock tự hết hạn sau 300ms vì không được release/renew

    loop Job chạy định kỳ (vd mỗi 5s)
        Reconciler->>DB: SELECT INVENTORY_DEDUCTION WHERE status=pending AND created_at < now() - lock_ttl_ms
        DB-->>Reconciler: Trả về deduction fencing_token=N+1, order_id=null, đã quá TTL mà vẫn pending
    end

    Reconciler->>DB: SELECT ORDER WHERE fencing_token_used=N+1
    DB-->>Reconciler: Không tìm thấy ORDER nào tương ứng, xác nhận đây là deduction dở dang thật sự

    Reconciler->>DB: UPDATE PRODUCT SET stock_quantity = stock_quantity + 1 (hoàn lại tồn kho bị kẹt)
    Reconciler->>DB: UPDATE INVENTORY_DEDUCTION SET status=refunded

    Note over Reconciler,DB: Tồn kho được hoàn lại đúng số lượng, sẵn sàng cho request tiếp theo, không bị mất vĩnh viễn do process crash giữa chừng
```

---

# Enhance sequence — Fencing token chặn ghi trừ kho từ process đã mất lock

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp process bị GC pause/network delay tưởng mình còn giữ lock. Đáp ứng yêu cầu 3: dùng fencing token gắn với lock để tránh process bị treo ghi trừ kho sau khi lock đã hết hạn và bị process khác giành mất — thao tác trừ kho với token cũ phải bị inventory service từ chối.

```mermaid
sequenceDiagram
    participant ProcA as Process A (bị GC pause)
    participant ProcB as Process B
    participant Lock as Redis Lock (lock:sku:SKU-123)
    participant InvSvc as Inventory Service
    participant DB as Database

    ProcA->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+1
    Lock-->>ProcA: Giành lock thành công

    ProcA->>ProcA: Bắt đầu xử lý trừ kho, bị GC pause dài giữa chừng

    Note over Lock: TTL 300ms hết hạn trong lúc ProcA bị pause, chưa kịp renew hay release

    ProcB->>Lock: SET lock:sku:SKU-123 NX PX 300ms, fencing_token=N+2
    Lock-->>ProcB: Giành lock thành công vì lock cũ đã hết hạn

    ProcB->>InvSvc: Trừ kho SKU-123 kèm fencing_token=N+2
    InvSvc->>InvSvc: Kiểm tra N+2 >= last_accepted_token (N+1), hợp lệ
    InvSvc->>DB: UPDATE PRODUCT SET stock_quantity = stock_quantity - 1
    InvSvc->>DB: INSERT ORDER (fencing_token_used=N+2, status=confirmed)
    InvSvc->>InvSvc: Cập nhật last_accepted_token=N+2
    InvSvc-->>ProcB: Mua thành công

    Note over ProcA: ProcA tỉnh lại sau GC pause, vẫn tưởng còn giữ lock với fencing_token=N+1
    ProcA->>InvSvc: Trừ kho SKU-123 kèm fencing_token=N+1 (đã trễ)

    InvSvc->>InvSvc: Kiểm tra N+1 < last_accepted_token (N+2), token đã cũ
    InvSvc-->>ProcA: Từ chối, "fencing token expired, lock đã đổi chủ, không thực hiện trừ kho"

    Note over InvSvc,DB: Nhờ vậy tồn kho không bị trừ 2 lần dù cả 2 process đều tưởng mình đang giữ lock hợp lệ
```

---

# Enhance sequence — Idempotency key chống trừ kho 2 lần khi client retry

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp client retry do timeout network. Đáp ứng yêu cầu 5: lock chỉ đảm bảo tuần tự xử lý chứ không đảm bảo request không bị gửi lặp, nên cần idempotency key gắn với request mua hàng, độc lập hoàn toàn với cơ chế lock, để đảm bảo retry không gây trừ kho 2 lần cho cùng 1 yêu cầu.

```mermaid
sequenceDiagram
    actor Client
    participant App as Order Service
    participant DB as Database (IDEMPOTENCY_RECORD)
    participant Lock as Redis Lock (lock:sku:SKU-123)

    Client->>App: Mua SKU-123 (idempotency_key=abc-123)
    App->>DB: SELECT IDEMPOTENCY_RECORD WHERE idempotency_key=abc-123
    DB-->>App: Không tìm thấy, đây là lần đầu

    App->>DB: INSERT IDEMPOTENCY_RECORD (idempotency_key=abc-123, result_status=processing)
    App->>Lock: SET lock:sku:SKU-123 NX PX 300ms
    Lock-->>App: Giành lock thành công

    App->>DB: Trừ kho, INSERT ORDER
    App->>Lock: DEL lock:sku:SKU-123

    App->>DB: UPDATE IDEMPOTENCY_RECORD SET result_status=success, order_id=...

    Note over Client,App: Response bị mất do network timeout, client không nhận được kết quả

    Client->>App: Retry, gửi lại request mua SKU-123 (cùng idempotency_key=abc-123)
    App->>DB: SELECT IDEMPOTENCY_RECORD WHERE idempotency_key=abc-123
    DB-->>App: Tìm thấy, result_status=success, order_id=...

    Note over App: Không lặp lại bước giành lock, không trừ kho lần nữa
    App-->>Client: Trả thẳng kết quả order đã có, "Mua thành công" (idempotent, không trừ kho lần 2)

    Note over App,DB: Idempotency check nằm hoàn toàn trước và độc lập với bước giành lock, nên bảo vệ được cả trường hợp lock đã release xong trước khi client kịp retry
```
