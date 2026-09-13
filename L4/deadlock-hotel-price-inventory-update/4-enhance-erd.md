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
