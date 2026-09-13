# Transaction checkout chạm nhiều bảng (đơn hàng, tồn kho, mã giảm giá) trên sàn e-commerce

**Hệ thống:** Một sàn e-commerce mà mỗi lượt checkout tạo 1 đơn hàng, trừ tồn kho của nhiều sản phẩm trong giỏ, và áp dụng mã giảm giá dùng 1 lần (giới hạn số lượt sử dụng).

**Vai trò của flow:** Transaction checkout chạm 3 nhóm bảng khác nhau (order, inventory nhiều dòng sản phẩm, coupon) trong cùng 1 lượt, thứ tự chạm các bảng này lại phụ thuộc thứ tự sản phẩm trong giỏ hàng của từng khách nên rất dễ tạo deadlock chéo giữa các đơn hàng cạnh tranh cùng lúc.

**Yêu cầu cụ thể:**
- Mô tả cụ thể: đơn hàng A có giỏ hàng gồm sản phẩm X rồi Y (khách thêm X trước), đơn hàng B có giỏ hàng gồm sản phẩm Y rồi X (khách thêm Y trước) — nếu code trừ tồn kho theo đúng thứ tự item xuất hiện trong giỏ của từng khách (không chuẩn hóa), A sẽ lock X trước rồi chờ lock Y, B sẽ lock Y trước rồi chờ lock X, tạo deadlock kinh điển; yêu cầu sort danh sách sản phẩm theo product_id tăng dần trước khi bắt đầu lock, bất kể thứ tự khách thêm vào giỏ.
- Chỉ ra rủi ro cross-table: nếu nhánh code validate mã giảm giá trước rồi mới trừ tồn kho ở 1 phần luồng (ví dụ đơn hàng dùng coupon), nhưng nhánh không dùng coupon lại trừ tồn kho trước rồi mới ghi nhận vào bảng order, thì 2 luồng có thể lock chéo giữa dòng coupon và các dòng inventory khi nhiều đơn hàng chạy song song — yêu cầu quy định 1 thứ tự lock cố định toàn cục cho mọi loại transaction checkout (ví dụ luôn: order row → inventory rows theo id tăng dần → coupon row cuối cùng), áp dụng nhất quán dù đơn có dùng coupon hay không.
- Mô tả cụ thể tình huống mùa sale: nhiều đơn hàng khác nhau chứa các sản phẩm trùng lặp một phần (đơn A có X, Y, Z; đơn B có Z, X) cùng chạy trong vài giây cao điểm — yêu cầu transaction bị deadlock phải tự động retry với backoff (tối đa 3 lần) và giới hạn thời gian tối đa cho 1 transaction checkout (ví dụ 2 giây) để tránh 1 đơn hàng bị treo kéo dài ảnh hưởng tới các đơn khác trong hàng đợi retry.
- Chọn isolation level `READ COMMITTED` kèm `SELECT ... FOR UPDATE` tường minh theo đúng thứ tự đã chuẩn hóa cho transaction checkout, giải thích vì sao dùng `SERIALIZABLE` cho toàn bộ transaction sẽ làm tăng abort rate không cần thiết trong giờ cao điểm khi nhiều đơn hàng không thực sự đụng chung sản phẩm nào.
- Viết test giả lập nhiều đơn hàng với thứ tự sản phẩm trong giỏ được hoán vị ngẫu nhiên (bao gồm cả trường hợp có/không dùng coupon), xác nhận không có transaction nào bị treo quá timeout quy định và invariant tồn kho + số lượt dùng coupon còn lại luôn đúng sau khi toàn bộ retry hoàn tất.

---

# Base ERD — Checkout chưa chuẩn hóa thứ tự lock

Đây là **base**: trạng thái hệ thống e-commerce *trước khi* áp dụng thứ tự lock cố định toàn cục. Suy luận từ đề bài, base đã có `PRODUCT`/`INVENTORY` (tồn kho theo sản phẩm), `ORDER`/`ORDER_ITEM` (đơn hàng và các dòng sản phẩm trong đơn), và `COUPON` (mã giảm giá dùng 1 lần, giới hạn số lượt). `ORDER_ITEM` có trường `added_sequence` ghi lại đúng thứ tự khách thêm sản phẩm vào giỏ — và trong base, code trừ tồn kho lại đi theo đúng thứ tự này (không chuẩn hóa), chính là nguồn gốc rủi ro deadlock nêu ở đề bài.

```mermaid
erDiagram
    PRODUCT ||--|| INVENTORY : "có tồn kho"
    ORDER ||--o{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "được đặt trong"
    ORDER }o--o| COUPON : "áp dụng (nếu có)"

    PRODUCT {
        string id PK
        string name
    }
    INVENTORY {
        string product_id PK
        int quantity
    }
    ORDER {
        string id PK
        string coupon_id FK "nullable"
        string status "pending|success|failed"
        datetime created_at
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string product_id FK
        int quantity
        int added_sequence "thứ tự khách thêm vào giỏ, chưa chuẩn hóa"
    }
    COUPON {
        string id PK
        string code
        int remaining_uses
    }
```

---

# Base sequence — Checkout (trừ tồn kho theo thứ tự giỏ hàng, gây deadlock chéo sản phẩm)

Đây là **base**, flow checkout ở trạng thái hiện tại: code trừ tồn kho theo đúng `added_sequence` (thứ tự khách thêm vào giỏ), không sort lại theo `product_id`. Đây chính là kịch bản deadlock kinh điển nêu ở yêu cầu 1 của đề bài — đơn hàng A thêm X trước Y, đơn hàng B thêm Y trước X, chạy đồng thời.

```mermaid
sequenceDiagram
    actor CustA as Khách A (giỏ hàng: X rồi Y)
    actor CustB as Khách B (giỏ hàng: Y rồi X)
    participant DB as Database
    participant InvX as INVENTORY (product X)
    participant InvY as INVENTORY (product Y)

    CustA->>DB: BEGIN Order A checkout
    CustB->>DB: BEGIN Order B checkout

    DB->>InvX: Order A: SELECT ... FOR UPDATE inventory X (theo added_sequence, X trước)
    InvX-->>DB: Lock granted cho Order A

    DB->>InvY: Order B: SELECT ... FOR UPDATE inventory Y (theo added_sequence, Y trước)
    InvY-->>DB: Lock granted cho Order B

    DB->>InvY: Order A: SELECT ... FOR UPDATE inventory Y - chờ lock
    Note over InvY: Inventory Y đang bị Order B giữ lock

    DB->>InvX: Order B: SELECT ... FOR UPDATE inventory X - chờ lock
    Note over InvX: Inventory X đang bị Order A giữ lock

    Note over DB,InvY: Order A chờ Order B, Order B chờ Order A, DEADLOCK
    DB-->>CustA: Lỗi deadlock, transaction rollback, không retry
    DB-->>CustB: Transaction còn lại tiếp tục commit thành công
```

---

# Base sequence — Checkout với/không coupon (thứ tự lock khác nhau giữa 2 nhánh)

Đây là **base**, flow checkout khi 2 nhánh code (đơn dùng coupon và đơn không dùng coupon) lock tài nguyên theo thứ tự khác nhau: nhánh dùng coupon validate/lock coupon trước rồi mới trừ tồn kho, nhánh không dùng coupon trừ tồn kho trước rồi mới ghi order. Đây là kịch bản rủi ro cross-table nêu ở yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor CustA as Khách A (đơn dùng coupon, có sản phẩm Z)
    actor CustB as Khách B (đơn không dùng coupon, có sản phẩm Z)
    participant DB as Database
    participant Cpn as COUPON row
    participant InvZ as INVENTORY (product Z)

    CustA->>DB: BEGIN Order A (có coupon)
    CustB->>DB: BEGIN Order B (không coupon)

    DB->>Cpn: Order A: SELECT ... FOR UPDATE coupon row (validate trước)
    Cpn-->>DB: Lock granted cho Order A

    DB->>InvZ: Order B: SELECT ... FOR UPDATE inventory Z (trừ tồn kho trước, chưa cần coupon)
    InvZ-->>DB: Lock granted cho Order B

    DB->>InvZ: Order A: SELECT ... FOR UPDATE inventory Z (trừ tồn kho sau khi coupon OK) - chờ lock
    Note over InvZ: Inventory Z đang bị Order B giữ lock

    DB->>Cpn: Order B: (giả sử đơn B sau đó cũng cần chạm coupon table để ghi thống kê) SELECT ... FOR UPDATE coupon row - chờ lock
    Note over Cpn: Coupon row đang bị Order A giữ lock

    Note over DB,InvZ: Order A chờ Order B (inventory Z), Order B chờ Order A (coupon), DEADLOCK chéo giữa 2 bảng khác nhau
    DB-->>CustA: Lỗi deadlock, transaction rollback
```

---

# Enhance ERD — Thứ tự lock cố định toàn cục, retry, timeout và deadlock logging

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `ORDER_ITEM` không còn dùng `added_sequence` để quyết định thứ tự lock (giữ lại chỉ để hiển thị UI), `ORDER` thêm các trường kiểm soát isolation/retry/timeout, và có thêm entity mới `DEADLOCK_LOG`, ứng trực tiếp với các yêu cầu:

- `ORDER.lock_order_product_ids` — đáp ứng yêu cầu 1 (sort product_id tăng dần trước khi lock, bất kể thứ tự giỏ hàng).
- `ORDER.isolation_level`, `ORDER.applied_global_lock_order` — đáp ứng yêu cầu 2 (thứ tự lock cố định toàn cục: order row → inventory rows tăng dần → coupon row cuối cùng, áp dụng nhất quán cả 2 nhánh có/không coupon) và yêu cầu 4 (READ COMMITTED + FOR UPDATE tường minh).
- `ORDER.retry_count`, `max_retry`, `timeout_ms` — đáp ứng yêu cầu 3 (retry tối đa 3 lần, timeout 2 giây/transaction).
- `DEADLOCK_LOG` (mới) — ghi chi tiết mỗi lần deadlock, phục vụ debug và test invariant ở yêu cầu 5.

```mermaid
erDiagram
    PRODUCT ||--|| INVENTORY : "có tồn kho"
    ORDER ||--o{ ORDER_ITEM : contains
    PRODUCT ||--o{ ORDER_ITEM : "được đặt trong"
    ORDER }o--o| COUPON : "áp dụng (nếu có)"
    ORDER ||--o{ DEADLOCK_LOG : "có thể phát sinh"

    PRODUCT {
        string id PK
        string name
    }
    INVENTORY {
        string product_id PK
        int quantity
    }
    ORDER {
        string id PK
        string coupon_id FK "nullable"
        string status "pending|success|failed"
        string isolation_level "READ_COMMITTED"
        string lock_order_product_ids "product_id đã sort tăng dần"
        boolean applied_global_lock_order "order -> inventory asc -> coupon cuối"
        int retry_count
        int max_retry "3"
        int timeout_ms "2000"
        datetime created_at
    }
    ORDER_ITEM {
        string id PK
        string order_id FK
        string product_id FK
        int quantity
        int added_sequence "chỉ dùng hiển thị UI, không dùng để lock"
    }
    COUPON {
        string id PK
        string code
        int remaining_uses
    }
    DEADLOCK_LOG {
        string id PK
        string order_id FK
        string blocking_order_id
        string locked_resource "vd: inventory product_id=Z, coupon row"
        string db_error_code
        int retry_attempt
        datetime occurred_at
    }
```

---

# Enhance sequence — Checkout (thứ tự lock cố định toàn cục: order → inventory tăng dần → coupon)

Đây là **enhance** của flow `checkout` đã có ở base, gộp cả 2 kịch bản base (deadlock chéo sản phẩm và deadlock chéo bảng coupon/inventory). So với base, mọi transaction checkout — dù có dùng coupon hay không — đều tuân theo đúng 1 thứ tự lock cố định: order row trước, rồi inventory rows theo `product_id` tăng dần (bỏ qua `added_sequence`), rồi coupon row lock sau cùng. Dùng isolation `READ COMMITTED` kèm `SELECT ... FOR UPDATE` tường minh theo đúng thứ tự này. Đáp ứng yêu cầu 1, 2 và 4 của đề bài.

```mermaid
sequenceDiagram
    actor CustA as Khách A (giỏ: X rồi Y, có coupon)
    actor CustB as Khách B (giỏ: Y rồi X, không coupon)
    participant DB as Database (isolation=READ COMMITTED)
    participant OrdRow as ORDER row
    participant InvX as INVENTORY (product X)
    participant InvY as INVENTORY (product Y)
    participant Cpn as COUPON row

    CustA->>DB: BEGIN Order A checkout
    CustA->>DB: Chuẩn hóa lock_order_product_ids = sort([X, Y]) = [X, Y]
    CustB->>DB: BEGIN Order B checkout
    CustB->>DB: Chuẩn hóa lock_order_product_ids = sort([Y, X]) = [X, Y]

    DB->>OrdRow: Order A: INSERT/lock order row
    DB->>InvX: Order A: SELECT ... FOR UPDATE inventory X (theo thứ tự cố định)
    InvX-->>DB: Lock granted cho Order A

    DB->>OrdRow: Order B: INSERT/lock order row
    DB->>InvX: Order B: SELECT ... FOR UPDATE inventory X - chờ lock
    Note over InvX: Order B luôn lock X trước Y (giống Order A), dù giỏ hàng của khách B thêm Y trước

    DB->>InvY: Order A: SELECT ... FOR UPDATE inventory Y (theo thứ tự cố định)
    InvY-->>DB: Lock granted cho Order A
    DB->>Cpn: Order A: SELECT ... FOR UPDATE coupon row (lock cuối cùng, sau inventory)
    Cpn-->>DB: Lock granted cho Order A
    DB->>DB: Order A: trừ tồn kho X, Y, trừ remaining_uses coupon, COMMIT
    DB->>InvX: Giải phóng lock inventory X

    DB->>InvX: Order B: nhận lock inventory X (vừa giải phóng)
    InvX-->>DB: Lock granted cho Order B
    DB->>InvY: Order B: SELECT ... FOR UPDATE inventory Y (theo thứ tự cố định)
    InvY-->>DB: Lock granted cho Order B
    Note over Cpn: Order B không dùng coupon nên bỏ qua bước lock coupon, nhưng thứ tự order->inventory vẫn nhất quán
    DB->>DB: Order B: trừ tồn kho X, Y, COMMIT

    Note over DB,Cpn: Vì cả 2 nhánh (có/không coupon) đều tuân theo cùng 1 thứ tự lock toàn cục, không còn vòng chờ chéo giữa các order cạnh tranh
```

---

# Enhance sequence — Retry với backoff và timeout tối đa cho checkout mùa sale

Đây là **enhance**, flow mới mô tả tình huống mùa sale: nhiều đơn hàng chứa sản phẩm trùng lặp một phần (đơn A có X, Y, Z; đơn B có Z, X) chạy trong vài giây cao điểm, vẫn có thể phát sinh deadlock dù đã chuẩn hóa thứ tự lock (do độ trễ, lock chờ chồng chéo nhiều đơn). Đáp ứng yêu cầu 3 của đề bài: retry tự động tối đa 3 lần với backoff, và giới hạn timeout 2 giây cho toàn bộ transaction checkout.

```mermaid
sequenceDiagram
    actor Cust as Khách hàng (checkout giờ cao điểm)
    participant App as Checkout Service
    participant DB as Database
    participant Log as DEADLOCK_LOG

    Cust->>App: Yêu cầu checkout
    App->>App: Bắt đầu đếm timeout_ms=2000 cho toàn bộ transaction
    App->>DB: BEGIN Order (retry_attempt=1), lock order->inventory asc->coupon
    DB-->>App: Lỗi deadlock (do nhiều order khác cùng chạm sản phẩm Z, X)
    App->>Log: Ghi DEADLOCK_LOG (order_id, blocking_order_id, locked_resource, retry_attempt=1)

    App->>App: Kiểm tra thời gian đã trôi qua < timeout_ms, backoff ngắn rồi retry
    App->>DB: BEGIN Order (retry_attempt=2)
    DB-->>App: Lỗi deadlock lần nữa
    App->>Log: Ghi DEADLOCK_LOG (retry_attempt=2)

    App->>App: Kiểm tra thời gian đã trôi qua vẫn < timeout_ms, backoff rồi retry
    App->>DB: BEGIN Order (retry_attempt=3)
    DB-->>App: Transaction thành công, COMMIT
    App-->>Cust: Checkout thành công

    Note over App,DB: Nếu tổng thời gian vượt quá timeout_ms=2000 trước khi hết 3 lần retry, App chủ động hủy transaction, trả lỗi "thử lại sau" thay vì để đơn hàng treo kéo dài, tránh ảnh hưởng các đơn khác trong hàng đợi retry
```

---

# Enhance sequence — Test giả lập checkout hoán vị giỏ hàng ngẫu nhiên, có/không coupon

Đây là **enhance**, flow mới mô tả kịch bản test tự động hoá. Đáp ứng yêu cầu 5 của đề bài: giả lập nhiều đơn hàng với thứ tự sản phẩm trong giỏ được hoán vị ngẫu nhiên (bao gồm cả có/không dùng coupon), xác nhận không transaction nào treo quá timeout và invariant tồn kho + số lượt dùng coupon còn lại luôn đúng sau khi mọi retry hoàn tất.

```mermaid
sequenceDiagram
    participant Test as Test Harness
    participant Orders as N đơn hàng (giỏ hàng hoán vị ngẫu nhiên, có/không coupon)
    participant DB as Database (lock order + retry + timeout đã áp dụng)
    participant Inv as INVENTORY (nhiều sản phẩm)
    participant Cpn as COUPON

    Test->>Inv: Đọc snapshot tồn kho ban đầu của từng sản phẩm
    Test->>Cpn: Đọc remaining_uses ban đầu của coupon
    Test->>Test: Tính expected_deduction dựa trên toàn bộ đơn hàng dự kiến thành công

    Test->>Orders: Khởi chạy đồng thời N đơn hàng, mỗi đơn có thứ tự sản phẩm trong giỏ ngẫu nhiên
    par Chạy song song trong vài giây cao điểm
        Orders->>DB: Đơn hàng có coupon (retry tối đa 3 lần, timeout 2 giây)
    and
        Orders->>DB: Đơn hàng không coupon (retry tối đa 3 lần, timeout 2 giây)
    end
    DB-->>Orders: Mỗi đơn hàng hoặc commit thành công trong timeout, hoặc bị hủy rõ ràng với lỗi "thử lại sau"

    Test->>Test: assert không đơn hàng nào có tổng thời gian xử lý > timeout_ms=2000
    Test->>Inv: Đọc tồn kho sau khi toàn bộ retry hoàn tất
    Test->>Cpn: Đọc remaining_uses sau khi toàn bộ retry hoàn tất
    Test->>Test: assert tồn kho == tồn kho ban đầu - tổng số lượng đã trừ bởi các đơn thành công
    Test->>Test: assert remaining_uses == remaining_uses ban đầu - số đơn thành công có dùng coupon

    Note over Test,DB: Vì thứ tự lock đã chuẩn hóa toàn cục, kết quả invariant không phụ thuộc vào thứ tự hoán vị giỏ hàng hay đơn nào chạy trước
```
