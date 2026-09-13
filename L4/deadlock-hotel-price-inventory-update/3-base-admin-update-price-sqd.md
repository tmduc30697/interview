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
