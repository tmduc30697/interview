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
