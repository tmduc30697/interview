# Giữ tồn kho khi seller và buyer thao tác đồng thời trên marketplace C2C

**Hệ thống:** Một marketplace C2C (kiểu Facebook Marketplace) cho phép seller đăng nhiều sản phẩm với số lượng, buyer có thể đặt mua và seller cũng có thể tự sửa/xóa số lượng bất cứ lúc nào.

**Vai trò của flow:** Flow reservation phải xử lý được xung đột giữa hành động buyer (đặt mua, giữ tồn kho) và hành động seller (tự sửa số lượng, ngừng bán) xảy ra đồng thời trên cùng 1 sản phẩm.

**Yêu cầu cụ thể:**
- Khi seller sửa số lượng tồn kho từ 5 xuống 2 đúng lúc có 3 buyer khác nhau đang giữ reservation (tổng 3 đơn vị đã reserve), quy định rõ hành vi: các reservation đã tạo trước khi seller sửa vẫn hợp lệ (không tự hủy), chỉ số lượng còn lại để bán mới bị giới hạn theo giá trị mới — không để số liệu "âm" (reserved vượt tồn kho mới) gây lỗi hiển thị.
- Khi seller bấm "Ngừng bán" (đóng sản phẩm) đúng lúc buyer đang ở giữa bước xác nhận đặt mua đã giữ reservation trước đó, request xác nhận của buyer phải được xử lý dựa trên trạng thái tại thời điểm reservation được tạo (buyer đã giữ hợp lệ thì vẫn được mua), còn buyer mới sau khi sản phẩm đã đóng không thể tạo reservation mới — mô tả rõ ranh giới thời điểm nào được tính là "trước/sau".
- 2 buyer cùng bấm "Mua ngay" cho sản phẩm chỉ còn 1 số lượng gần như đồng thời: chỉ 1 người giữ được reservation, thiết kế bằng update nguyên tử trên cột tồn kho khả dụng, có test giả lập concurrency xác nhận không có 2 reservation cùng tồn tại cho 1 đơn vị hàng.
- Reservation của buyer phải có TTL ngắn hơn ở marketplace C2C so với e-commerce chính thức (vì tốc độ trao đổi/chat trước khi chốt mua có thể chậm), quy định giá trị TTL và cách gia hạn khi buyer và seller đang chat để thống nhất giao dịch.
- Có audit log cho mọi thay đổi tồn kho (ai sửa, khi nào, giá trị trước/sau) để xử lý tranh chấp khi buyer khiếu nại "đã đặt được hàng nhưng sau đó bị báo hết".

---

# ERD - Base (trước khi xử lý concurrency cho reservation C2C)

Đây là **base**: mô hình dữ liệu tối thiểu của một marketplace C2C, nơi seller đăng sản phẩm với số lượng, buyer đặt mua để giữ chỗ. Chỉ giữ `Seller`, `Buyer`, `Product`, `Reservation` - đủ để thấy vấn đề: `Product.quantity` là một con số đơn giản, còn `Reservation` chưa có TTL rõ ràng hay cơ chế atomic, chưa có audit log - đúng những gì enhance sẽ bổ sung.

```mermaid
erDiagram
    SELLER ||--o{ PRODUCT : lists
    BUYER ||--o{ RESERVATION : creates
    PRODUCT ||--o{ RESERVATION : reserved_in

    SELLER {
        string id PK
        string name
    }

    BUYER {
        string id PK
        string name
    }

    PRODUCT {
        string id PK
        string seller_id FK
        string title
        int quantity
        string status
        datetime created_at
    }

    RESERVATION {
        string id PK
        string product_id FK
        string buyer_id FK
        int quantity
        string status
        datetime created_at
    }
```

---

# Sequence - Base - Flow "create-reservation"

Đây là **base**: cách buyer đặt mua (giữ reservation) trước khi có xử lý concurrency, dùng read-modify-write đơn giản trên `quantity`. Flow này được chọn vì nó chính là nơi race condition xảy ra khi enhance yêu cầu 2 buyer bấm "Mua ngay" gần như đồng thời cho sản phẩm chỉ còn 1 số lượng (yêu cầu 3).

```mermaid
sequenceDiagram
    actor Buyer1 as Buyer A
    actor Buyer2 as Buyer B
    participant App as Marketplace App
    participant DB as Database

    Note over Buyer1,Buyer2: Sản phẩm còn quantity = 1, 2 buyer bấm Mua ngay gần như đồng thời

    Buyer1->>App: Mua ngay sản phẩm X
    App->>DB: đọc quantity hiện tại (đọc = 1)

    Buyer2->>App: Mua ngay sản phẩm X
    App->>DB: đọc quantity hiện tại (đọc = 1, chưa thấy thay đổi của Buyer A)

    App-->>App: (Buyer A) kiểm tra quantity > 0, hợp lệ
    App->>DB: ghi quantity = 0, tạo reservation cho Buyer A

    App-->>App: (Buyer B) kiểm tra quantity > 0 dựa trên giá trị đã đọc trước đó, cũng hợp lệ
    App->>DB: ghi quantity = 0, tạo reservation cho Buyer B

    Note over DB: Kết quả sai: 2 reservation cùng tồn tại cho 1 đơn vị hàng
```

---

# Sequence - Base - Flow "update-quantity"

Đây là **base**: cách seller sửa số lượng tồn kho trước khi có quy tắc phối hợp với reservation đang tồn tại - đơn giản là ghi đè `quantity` mới, không xét tới các reservation đã giữ trước đó. Flow này được chọn vì nó là tiền đề cho yêu cầu 1 (seller sửa số lượng khi buyer đang giữ reservation) và yêu cầu 5 (chưa có audit log nào ghi lại thay đổi).

```mermaid
sequenceDiagram
    actor Seller
    participant App as Marketplace App
    participant DB as Database

    Note over DB: quantity hiện tại = 5, đã có 3 reservation active (tổng 3 đơn vị)

    Seller->>App: sửa số lượng tồn kho từ 5 xuống 2
    App->>DB: ghi đè quantity = 2

    Note over DB: Không có ghi chú ai sửa, khi nào, giá trị trước/sau\nKhông xét reservation đang active, dữ liệu có thể hiển thị "âm"
```

---

# Sequence - Base - Flow "close-product"

Đây là **base**: cách seller ngừng bán một sản phẩm trước khi có quy định ranh giới thời điểm rõ ràng với buyer đang xác nhận đặt mua. Flow này được chọn vì nó là tiền đề cho yêu cầu 2 - đề bài yêu cầu định nghĩa rõ "trước/sau" thời điểm đóng sản phẩm, còn ở base thì hành vi mơ hồ, tùy thời điểm request buyer chạm tới app trước hay sau khi cờ status đổi.

```mermaid
sequenceDiagram
    actor Buyer
    actor Seller
    participant App as Marketplace App
    participant DB as Database

    Note over Buyer: Buyer đã tạo reservation trước đó, đang ở bước xác nhận đặt mua

    Seller->>App: Ngừng bán sản phẩm X
    App->>DB: ghi status = closed

    Buyer->>App: xác nhận đặt mua (dùng reservation đã giữ)
    App->>DB: kiểm tra status sản phẩm

    alt Request buyer tới trước khi status đổi
        App-->>Buyer: xác nhận thành công
    else Request buyer tới sau khi status đổi
        App-->>Buyer: từ chối vì sản phẩm đã đóng, dù reservation vẫn còn hiệu lực
    end

    Note over App,DB: Không có quy tắc rõ ràng nào phân biệt 2 trường hợp trên,\nkết quả phụ thuộc thời điểm request chạm server
```

---

# ERD - Enhance (reservation an toàn cho concurrency C2C)

Đây là **enhance**: mô hình dữ liệu sau khi thêm cơ chế xử lý concurrency giữa seller và buyer. So với base, các thay đổi là:

- `PRODUCT` có thêm `reserved_count` (tổng số lượng đang bị giữ bởi reservation active, tách khỏi `quantity` để không cần hủy reservation cũ khi seller giảm số lượng - yêu cầu 1), `version` (dùng cho update nguyên tử/optimistic lock trên tồn kho khả dụng - yêu cầu 3), và `closed_at` (mốc thời gian chính xác để phân biệt reservation "trước/sau" khi ngừng bán - yêu cầu 2).
- `RESERVATION` có thêm `expires_at`, `extended_count` (TTL ngắn và cơ chế gia hạn khi buyer/seller đang chat - yêu cầu 4), và `status` mở rộng (`active` / `confirmed` / `expired` / `cancelled`) để phân biệt trạng thái tại thời điểm reservation được tạo.
- `INVENTORY_AUDIT_LOG` (mới): 1 bản ghi cho mỗi lần `quantity` hoặc `reserved_count` thay đổi, lưu ai sửa, khi nào, giá trị trước/sau - phục vụ tra soát tranh chấp (yêu cầu 5).

```mermaid
erDiagram
    SELLER ||--o{ PRODUCT : lists
    BUYER ||--o{ RESERVATION : creates
    PRODUCT ||--o{ RESERVATION : reserved_in
    PRODUCT ||--o{ INVENTORY_AUDIT_LOG : has_history

    SELLER {
        string id PK
        string name
    }

    BUYER {
        string id PK
        string name
    }

    PRODUCT {
        string id PK
        string seller_id FK
        string title
        int quantity
        int reserved_count
        int version
        string status
        datetime closed_at
        datetime created_at
    }

    RESERVATION {
        string id PK
        string product_id FK
        string buyer_id FK
        int quantity
        string status
        datetime created_at
        datetime expires_at
        int extended_count
        datetime confirmed_at
    }

    INVENTORY_AUDIT_LOG {
        string id PK
        string product_id FK
        string changed_by_type
        string changed_by_id
        string field_changed
        int old_value
        int new_value
        string reason
        datetime changed_at
    }
```

---

# Sequence - Enhance - Flow "create-reservation"

Đây là **enhance**, cùng flow `create-reservation` như ở base nhưng thay update read-modify-write bằng update nguyên tử có điều kiện trên `reserved_count`, đáp ứng yêu cầu 3: 2 buyer bấm "Mua ngay" gần như đồng thời cho sản phẩm chỉ còn 1 số lượng khả dụng thì chỉ 1 người giữ được reservation. Flow cũng gán `expires_at` theo TTL ngắn của C2C ngay khi tạo, chuẩn bị cho yêu cầu 4.

```mermaid
sequenceDiagram
    actor Buyer1 as Buyer A
    actor Buyer2 as Buyer B
    participant App as Marketplace App
    participant DB as Database

    Note over DB: quantity = 5, reserved_count = 4, còn khả dụng = 1

    par Buyer A bấm Mua ngay
        Buyer1->>App: Mua ngay sản phẩm X
        App->>DB: UPDATE product SET reserved_count = reserved_count + 1\nWHERE id = X AND quantity - reserved_count >= 1
        DB-->>App: 1 dòng bị ảnh hưởng, thành công
        App->>DB: tạo reservation cho Buyer A, expires_at = now + TTL ngắn
        App-->>Buyer1: giữ chỗ thành công
    and Buyer B bấm Mua ngay gần như cùng lúc
        Buyer2->>App: Mua ngay sản phẩm X
        App->>DB: UPDATE product SET reserved_count = reserved_count + 1\nWHERE id = X AND quantity - reserved_count >= 1
        DB-->>App: 0 dòng bị ảnh hưởng, điều kiện không còn đúng
        App-->>Buyer2: hết hàng, không tạo reservation
    end

    Note over DB: Đảm bảo không có 2 reservation cùng tồn tại cho 1 đơn vị hàng,\nxác nhận bằng test giả lập concurrency chạy nhiều lần
```

---

# Sequence - Enhance - Flow "update-quantity"

Đây là **enhance**, cùng flow `update-quantity` như ở base nhưng thay đổi hành vi cốt lõi, đáp ứng yêu cầu 1 và yêu cầu 5: khi seller sửa `quantity` từ 5 xuống 2 lúc đang có 3 reservation active (tổng 3 đơn vị), các reservation cũ vẫn hợp lệ - không bị hủy tự động; chỉ số lượng còn lại để bán mới (`quantity - reserved_count`) bị giới hạn và được kẹp về 0 thay vì hiển thị số âm. Mọi thay đổi đều ghi audit log.

```mermaid
sequenceDiagram
    actor Seller
    participant App as Marketplace App
    participant DB as Database
    participant AuditLog as Inventory Audit Log

    Note over DB: quantity = 5, reserved_count = 3 (3 reservation active không đổi)

    Seller->>App: sửa số lượng tồn kho từ 5 xuống 2
    App->>DB: UPDATE product SET quantity = 2 WHERE id = X
    App->>AuditLog: ghi log, changed_by = Seller, field = quantity, old_value = 5, new_value = 2

    Note over App,DB: 3 reservation active vẫn giữ nguyên, không tự hủy

    App-->>App: tính số lượng còn lại để bán = max(0, quantity - reserved_count)\n= max(0, 2 - 3) = 0

    App-->>Seller: hiển thị "còn lại để bán: 0", không hiển thị số âm

    Note over App: Buyer mới truy cập sản phẩm cũng thấy còn lại để bán = 0,\nkhông thể tạo reservation mới cho tới khi reservation cũ giải phóng
```

---

# Sequence - Enhance - Flow "close-product"

Đây là **enhance**, cùng flow `close-product` như ở base nhưng thay đổi hành vi cốt lõi, đáp ứng yêu cầu 2: ranh giới "trước/sau" được định nghĩa rõ bằng `reservation.created_at` so với `product.closed_at`, không phụ thuộc vào thời điểm request của buyer chạm tới server. Buyer đã giữ reservation hợp lệ trước mốc đóng vẫn được mua, buyer mới sau mốc đó không thể tạo reservation.

```mermaid
sequenceDiagram
    actor BuyerOld as Buyer đã giữ reservation trước đó
    actor BuyerNew as Buyer mới
    actor Seller
    participant App as Marketplace App
    participant DB as Database
    participant AuditLog as Inventory Audit Log

    Note over BuyerOld: reservation.created_at = 10:00:00, status = active

    Seller->>App: Ngừng bán sản phẩm X lúc 10:00:05
    App->>DB: UPDATE product SET status = closed, closed_at = 10:00:05
    App->>AuditLog: ghi log, changed_by = Seller, field = status, old_value = active, new_value = closed

    BuyerOld->>App: xác nhận đặt mua lúc 10:00:10 (dùng reservation tạo lúc 10:00:00)
    App->>DB: kiểm tra reservation.created_at (10:00:00) so với product.closed_at (10:00:05)
    Note over App: reservation.created_at < closed_at => hợp lệ theo mốc thời điểm tạo
    App->>DB: reservation.status = confirmed
    App-->>BuyerOld: xác nhận đặt mua thành công

    BuyerNew->>App: cố tạo reservation mới cho sản phẩm X lúc 10:00:20
    App->>DB: kiểm tra product.status
    Note over App: product.status = closed => không cho tạo reservation mới
    App-->>BuyerNew: từ chối, sản phẩm đã ngừng bán
```

---

# Sequence - Enhance - Flow "reservation-ttl-extend"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 4: reservation ở marketplace C2C có TTL ngắn hơn e-commerce chính thức (vì buyer/seller cần thời gian chat để thống nhất giao dịch), và có cơ chế gia hạn khi 2 bên vẫn đang trao đổi thay vì để reservation tự hết hạn giữa chừng.

```mermaid
sequenceDiagram
    actor Buyer
    actor Seller
    participant Chat as Kênh chat
    participant App as Marketplace App
    participant DB as Database
    participant Scheduler as TTL Scheduler

    Note over DB: reservation tạo lúc 10:00:00, TTL C2C ngắn = 15 phút,\nexpires_at = 10:15:00 (ngắn hơn nhiều so với TTL e-commerce)

    Buyer->>Chat: nhắn tin hỏi seller về giao dịch
    Seller->>Chat: phản hồi, 2 bên đang thống nhất giá/giao hàng

    Scheduler-->>Scheduler: tới gần expires_at (còn 2 phút), phát hiện reservation sắp hết hạn

    alt Buyer hoặc Seller còn hoạt động trong kênh chat
        App->>DB: gia hạn expires_at thêm 1 khoảng TTL, extended_count += 1
        App-->>Buyer: reservation được gia hạn, tiếp tục giữ chỗ
    else Không có hoạt động chat nào trong TTL
        Scheduler->>DB: UPDATE reservation SET status = expired WHERE id = X AND status = active
        App->>DB: reserved_count -= 1, giải phóng lại số lượng khả dụng
        App-->>Buyer: thông báo reservation đã hết hạn, cần đặt lại nếu vẫn muốn mua
    end

    Note over DB: extended_count có giới hạn tối đa để tránh giữ chỗ vô thời hạn\nkhi 2 bên chat nhưng không chốt mua
```
