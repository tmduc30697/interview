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
