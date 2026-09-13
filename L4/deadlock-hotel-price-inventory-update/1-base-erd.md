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
