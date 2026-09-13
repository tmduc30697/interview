# Đặt phòng khách sạn với cập nhật giá và tồn phòng cùng lúc

**Hệ thống:** Một nền tảng đặt phòng khách sạn, admin khách sạn có thể cập nhật giá/số phòng trống trong khi khách đang đặt phòng.

**Vai trò của flow:** Transaction đặt phòng (trừ tồn phòng) và transaction admin cập nhật giá/tồn phòng cùng chạm vào bảng `room_inventory`, cần chọn isolation level và thứ tự lock hợp lý để không deadlock và không đọc dữ liệu giá sai (stale).

**Yêu cầu cụ thể:**
- Định nghĩa rõ transaction đặt phòng phải `SELECT ... FOR UPDATE` trên đúng các row (room_type, date) cần trừ tồn, không lock nguyên bảng.
- Nếu một request đặt phòng theo thứ tự ngày [10/9, 11/9, 12/9] và request khác đặt theo thứ tự [12/9, 11/9, 10/9] (đặt nhiều đêm cùng lúc), phải chuẩn hóa thứ tự lock theo khóa chính tăng dần trước khi lock để loại bỏ deadlock giữa 2 booking đa-ngày chồng lấn.
- So sánh và chọn isolation level: dùng `READ COMMITTED` + lock tường minh cho phần trừ tồn, tránh dùng `SERIALIZABLE` toàn bộ vì sẽ làm tăng abort rate không cần thiết ở phần đọc giá hiển thị cho khách đang xem trang.
- Khi admin sửa giá đúng lúc khách đang giữ lock trừ tồn cho phòng đó, request admin phải chờ hoặc bị timeout rõ ràng (không treo vô hạn) — quy định timeout cụ thể (ví dụ 3 giây) và hành vi khi timeout (báo lỗi "thử lại sau").
- Viết test dựng 2 transaction giả lập chồng lấn lock theo thứ tự ngược nhau, xác nhận hệ thống tự phát hiện/rollback một bên và request còn lại thành công.

---

# Base ERD — Đặt phòng và cập nhật giá chưa có chiến lược lock rõ ràng

Đây là **base**: trạng thái nền tảng đặt phòng khách sạn *trước khi* áp dụng `SELECT ... FOR UPDATE` tường minh, chuẩn hóa thứ tự lock và timeout cho admin. Suy luận từ đề bài, base đã có `ROOM_TYPE`, `ROOM_INVENTORY` (tồn phòng và giá theo từng `room_type` + `date`), `BOOKING` và `BOOKING_ITEM` (mỗi đêm khách đặt là 1 dòng, theo đúng thứ tự khách chọn ngày, chưa chuẩn hóa). Base chưa có ràng buộc nào về isolation level hay timeout khi admin và khách cùng chạm 1 dòng `ROOM_INVENTORY`.

```mermaid
erDiagram
    ROOM_TYPE ||--o{ ROOM_INVENTORY : "tồn phòng theo ngày"
    BOOKING ||--o{ BOOKING_ITEM : contains
    ROOM_INVENTORY ||--o{ BOOKING_ITEM : "trừ tồn từ"

    ROOM_TYPE {
        string id PK
        string hotel_id
        string name
    }
    ROOM_INVENTORY {
        string id PK
        string room_type_id FK
        date date
        int available_count
        decimal price
    }
    BOOKING {
        string id PK
        string guest_id
        string status "pending|confirmed|failed"
        datetime created_at
    }
    BOOKING_ITEM {
        string id PK
        string booking_id FK
        string room_inventory_id FK
        int added_sequence "thứ tự khách chọn ngày, chưa chuẩn hóa"
    }
```

---

# Base sequence — Book room (lock theo thứ tự ngày khách chọn, gây deadlock đặt đa-ngày chồng lấn)

Đây là **base**, flow đặt phòng ở trạng thái hiện tại: transaction lock các dòng `ROOM_INVENTORY` theo đúng thứ tự ngày mà request gửi lên (không sort lại). Đây chính là kịch bản deadlock nêu ở yêu cầu 2 của đề bài — 1 request đặt theo thứ tự [10/9, 11/9, 12/9], request khác đặt cùng room_type theo thứ tự [12/9, 11/9, 10/9].

```mermaid
sequenceDiagram
    actor GuestA as Khách A (đặt thứ tự 10/9, 11/9, 12/9)
    actor GuestB as Khách B (đặt thứ tự 12/9, 11/9, 10/9)
    participant DB as Database
    participant Inv10 as ROOM_INVENTORY (10/9)
    participant Inv12 as ROOM_INVENTORY (12/9)

    GuestA->>DB: BEGIN Booking A
    GuestB->>DB: BEGIN Booking B

    DB->>Inv10: Booking A: SELECT ... FOR UPDATE room_inventory (10/9, theo thứ tự request)
    Inv10-->>DB: Lock granted cho Booking A

    DB->>Inv12: Booking B: SELECT ... FOR UPDATE room_inventory (12/9, theo thứ tự request)
    Inv12-->>DB: Lock granted cho Booking B

    DB->>Inv12: Booking A: SELECT ... FOR UPDATE room_inventory (12/9) - chờ lock
    Note over Inv12: Dòng 12/9 đang bị Booking B giữ lock

    DB->>Inv10: Booking B: SELECT ... FOR UPDATE room_inventory (10/9) - chờ lock
    Note over Inv10: Dòng 10/9 đang bị Booking A giữ lock

    Note over DB,Inv12: Booking A chờ Booking B, Booking B chờ Booking A, DEADLOCK
    DB-->>GuestA: Lỗi deadlock, transaction rollback
    DB-->>GuestB: Transaction còn lại commit thành công
```

---

# Base sequence — Admin update price (không timeout, có thể treo vô hạn hoặc đọc giá stale)

Đây là **base**, flow admin cập nhật giá/tồn phòng ở trạng thái hiện tại: khi khách đang giữ lock trừ tồn cho đúng phòng/ngày đó, request admin không có timeout rõ ràng nên có thể chờ vô hạn, hoặc nếu không lock tường minh thì đọc phải giá cũ (stale) rồi ghi đè sai. Đây là tiền đề cho yêu cầu 1 và 4 của đề bài.

```mermaid
sequenceDiagram
    actor Guest as Khách (đang đặt phòng, giữ lock)
    actor Admin as Admin khách sạn
    participant DB as Database
    participant Inv as ROOM_INVENTORY (room_type X, ngày 10/9)

    Guest->>DB: BEGIN Booking, SELECT ... FOR UPDATE room_inventory (X, 10/9)
    Inv-->>DB: Lock granted cho Guest

    Admin->>DB: UPDATE giá phòng room_inventory (X, 10/9)
    DB->>Inv: Yêu cầu ghi vào dòng đang bị Guest giữ lock - chờ lock
    Note over Inv,DB: Không có timeout, request admin bị treo cho tới khi Guest hoàn tất transaction (có thể rất lâu nếu Guest gặp sự cố mạng)

    Guest->>DB: Guest hoàn tất, COMMIT, giải phóng lock
    DB->>Inv: Admin nhận lock, UPDATE giá thành công (chậm, không rõ khi nào)
    DB-->>Admin: Không có phản hồi trung gian trong lúc chờ, trải nghiệm admin bị treo
```

---

# Enhance ERD — Chuẩn hóa thứ tự lock, isolation level tường minh và timeout cho admin

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, `BOOKING` thêm các trường kiểm soát isolation/thứ tự lock, và có thêm entity mới `ADMIN_PRICE_UPDATE` để quản lý timeout khi admin sửa giá, ứng trực tiếp với các yêu cầu:

- `BOOKING.lock_order_inventory_ids`, `isolation_level` — đáp ứng yêu cầu 1 (chỉ `SELECT ... FOR UPDATE` đúng row cần trừ tồn), yêu cầu 2 (chuẩn hóa thứ tự lock theo khóa chính tăng dần) và yêu cầu 3 (READ COMMITTED thay vì SERIALIZABLE toàn bộ).
- `ADMIN_PRICE_UPDATE` (mới) — ghi nhận `timeout_ms` và `status` khi admin sửa giá đúng lúc khách giữ lock, đáp ứng yêu cầu 4.
- `DEADLOCK_TEST_LOG` (mới) — phục vụ test giả lập 2 transaction chồng lấn ngược thứ tự, đáp ứng yêu cầu 5.

```mermaid
erDiagram
    ROOM_TYPE ||--o{ ROOM_INVENTORY : "tồn phòng theo ngày"
    BOOKING ||--o{ BOOKING_ITEM : contains
    ROOM_INVENTORY ||--o{ BOOKING_ITEM : "trừ tồn từ"
    ROOM_INVENTORY ||--o{ ADMIN_PRICE_UPDATE : "có thể bị sửa giá"
    BOOKING ||--o{ DEADLOCK_TEST_LOG : "có thể phát sinh"

    ROOM_TYPE {
        string id PK
        string hotel_id
        string name
    }
    ROOM_INVENTORY {
        string id PK
        string room_type_id FK
        date date
        int available_count
        decimal price
    }
    BOOKING {
        string id PK
        string guest_id
        string status "pending|confirmed|failed"
        string isolation_level "READ_COMMITTED"
        string lock_order_inventory_ids "room_inventory_id đã sort tăng dần"
        datetime created_at
    }
    BOOKING_ITEM {
        string id PK
        string booking_id FK
        string room_inventory_id FK
        int added_sequence "chỉ dùng hiển thị UI, không dùng để lock"
    }
    ADMIN_PRICE_UPDATE {
        string id PK
        string room_inventory_id FK
        decimal requested_price
        int timeout_ms "3000"
        string status "success|timeout"
        datetime requested_at
    }
    DEADLOCK_TEST_LOG {
        string id PK
        string booking_id FK
        string blocking_booking_id
        string locked_resource "vd: room_inventory_id=..."
        string outcome "rolled_back|committed"
        datetime occurred_at
    }
```

---

# Enhance sequence — Book room (chỉ FOR UPDATE đúng row, lock theo khóa chính tăng dần)

Đây là **enhance** của flow `book-room` đã có ở base. So với base (lock theo thứ tự request, gây deadlock), nay transaction chỉ `SELECT ... FOR UPDATE` đúng các row (room_type, date) cần trừ tồn — không lock nguyên bảng — và luôn sort `room_inventory_id` tăng dần trước khi lock, bất kể thứ tự khách chọn ngày. Đáp ứng yêu cầu 1, 2 và 3 của đề bài.

```mermaid
sequenceDiagram
    actor GuestA as Khách A (đặt thứ tự 10/9, 11/9, 12/9)
    actor GuestB as Khách B (đặt thứ tự 12/9, 11/9, 10/9)
    participant DB as Database (isolation=READ COMMITTED)
    participant Inv10 as ROOM_INVENTORY (10/9)
    participant Inv11 as ROOM_INVENTORY (11/9)
    participant Inv12 as ROOM_INVENTORY (12/9)

    GuestA->>DB: BEGIN Booking A
    GuestA->>DB: Chuẩn hóa lock_order_inventory_ids = sort([10/9, 11/9, 12/9])
    GuestB->>DB: BEGIN Booking B
    GuestB->>DB: Chuẩn hóa lock_order_inventory_ids = sort([12/9, 11/9, 10/9]) = giống Booking A

    DB->>Inv10: Booking A: SELECT ... FOR UPDATE room_inventory (10/9, đúng row cần trừ tồn)
    Inv10-->>DB: Lock granted cho Booking A

    DB->>Inv10: Booking B: SELECT ... FOR UPDATE room_inventory (10/9) - chờ lock
    Note over Inv10: Booking B luôn lock 10/9 trước, giống Booking A, dù khách B chọn ngày theo thứ tự ngược lại

    DB->>Inv11: Booking A: SELECT ... FOR UPDATE room_inventory (11/9)
    Inv11-->>DB: Lock granted cho Booking A
    DB->>Inv12: Booking A: SELECT ... FOR UPDATE room_inventory (12/9)
    Inv12-->>DB: Lock granted cho Booking A
    DB->>DB: Booking A: trừ tồn 3 ngày, COMMIT
    DB->>Inv10: Giải phóng lock 10/9

    DB->>Inv10: Booking B: nhận lock (vừa giải phóng)
    Inv10-->>DB: Lock granted cho Booking B
    DB->>Inv11: Booking B: SELECT ... FOR UPDATE room_inventory (11/9)
    Inv11-->>DB: Lock granted cho Booking B
    DB->>Inv12: Booking B: SELECT ... FOR UPDATE room_inventory (12/9)
    Inv12-->>DB: Lock granted cho Booking B
    DB->>DB: Booking B: trừ tồn 3 ngày, COMMIT

    Note over DB,Inv12: Vì cả 2 booking đều lock theo cùng thứ tự tăng dần, không còn vòng chờ chéo, không cần SERIALIZABLE toàn bộ vẫn loại trừ được deadlock
```

---

# Enhance sequence — Admin update price (timeout rõ ràng, không treo vô hạn)

Đây là **enhance** của flow `admin-update-price` đã có ở base. So với base (không timeout, có thể treo vô hạn), nay request admin có `timeout_ms=3000` khi chờ lock của khách đang trừ tồn cùng row, và trả lỗi rõ ràng "thử lại sau" khi timeout thay vì treo vô thời hạn. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor Guest as Khách (đang đặt phòng, giữ lock)
    actor Admin as Admin khách sạn
    participant DB as Database
    participant Inv as ROOM_INVENTORY (room_type X, ngày 10/9)
    participant Upd as ADMIN_PRICE_UPDATE

    Guest->>DB: BEGIN Booking, SELECT ... FOR UPDATE room_inventory (X, 10/9)
    Inv-->>DB: Lock granted cho Guest

    Admin->>DB: UPDATE giá room_inventory (X, 10/9)
    DB->>Upd: Tạo ADMIN_PRICE_UPDATE (timeout_ms=3000, status=pending)
    DB->>Inv: Yêu cầu ghi vào dòng đang bị Guest giữ lock - chờ lock, đếm timeout

    alt Guest hoàn tất trước 3 giây
        Guest->>DB: COMMIT, giải phóng lock
        DB->>Inv: Admin nhận lock, UPDATE giá thành công
        DB->>Upd: Cập nhật status=success
        DB-->>Admin: Cập nhật giá thành công
    else Guest chưa hoàn tất sau 3 giây
        DB->>Upd: Cập nhật status=timeout
        DB-->>Admin: Báo lỗi rõ ràng "thử lại sau", không treo vô hạn
    end
```

---

# Enhance sequence — Test 2 booking chồng lấn lock theo thứ tự ngược nhau

Đây là **enhance**, flow mới mô tả kịch bản test tự động hoá. Đáp ứng yêu cầu 5 của đề bài: dựng 2 transaction giả lập chồng lấn lock theo thứ tự ngày ngược nhau, cùng tranh chấp phòng cuối cùng còn trống, xác nhận hệ thống tự phát hiện/rollback một bên (do hết phòng khi đến lượt, không phải do deadlock — deadlock đã bị loại trừ nhờ chuẩn hóa thứ tự lock) và request còn lại thành công.

```mermaid
sequenceDiagram
    participant Test as Test Harness
    participant BookA as Booking A (yêu cầu [10/9, 11/9, 12/9], cần 1 phòng cuối cùng)
    participant BookB as Booking B (yêu cầu [12/9, 11/9, 10/9], cũng cần 1 phòng cuối cùng)
    participant DB as Database (lock order đã chuẩn hóa)
    participant Log as DEADLOCK_TEST_LOG

    Test->>DB: Setup available_count=1 cho cả 3 ngày (10/9, 11/9, 12/9)
    Test->>BookA: Khởi chạy Booking A cùng lúc với Booking B
    Test->>BookB: Khởi chạy Booking B cùng lúc với Booking A

    par Chạy song song, cùng chạm 3 ngày trùng nhau
        BookA->>DB: BEGIN, lock theo thứ tự đã chuẩn hóa [10/9, 11/9, 12/9]
    and
        BookB->>DB: BEGIN, lock theo thứ tự đã chuẩn hóa [10/9, 11/9, 12/9] (giống Booking A dù request ngược)
    end

    Note over DB: Vì cùng thứ tự lock, Booking B chỉ phải chờ Booking A giải phóng lock, không có vòng chờ chéo -> không deadlock
    DB->>DB: Booking A vào trước, kiểm tra available_count=1 đủ, trừ còn 0, COMMIT
    DB->>DB: Booking B vào sau, kiểm tra available_count=0 không đủ, tự rollback theo logic nghiệp vụ (hết phòng)
    DB-->>Test: Booking A commit thành công, Booking B rollback với lỗi "hết phòng"

    Test->>Log: Ghi DEADLOCK_TEST_LOG (booking_id=B, blocking_booking_id=A, locked_resource=room_inventory 3 ngày, outcome=rolled_back)
    Test->>Test: assert Booking A thành công, Booking B bị rollback rõ ràng (không phải do deadlock), available_count cuối cùng = 0
```
