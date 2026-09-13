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
