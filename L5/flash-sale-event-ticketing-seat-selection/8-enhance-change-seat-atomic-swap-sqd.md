# Sequence - Enhance: Đổi ghế đã giữ sang ghế khác (transaction atomic)

Đây là flow **enhance hoàn toàn mới**, mô tả khách đã giữ ghế A12 nhưng đổi ý muốn chuyển sang ghế A13. Thao tác nhả ghế cũ và giữ ghế mới phải nằm trong 1 transaction atomic — không được để khoảng hở giữa 2 bước khiến ghế mới bị người khác cướp mất ngay sau khi ghế cũ vừa nhả, và nếu bước giữ ghế mới thất bại thì toàn bộ transaction rollback, ghế cũ vẫn được giữ nguyên chứ không bị nhả oan. Đáp ứng **yêu cầu 4** của đề bài.

```mermaid
sequenceDiagram
    participant U as Khách hàng (đang giữ A12)
    participant API as Ticketing API
    participant DB as Database

    Note over DB: SEAT A12: status=held, held_by_user_id=U / SEAT A13: status=available
    U->>API: Đổi từ ghế A12 sang ghế A13
    API->>DB: BEGIN TRANSACTION
    API->>DB: UPDATE SEAT SET status='held', held_by_user_id=U, held_until=now+5' WHERE seat_id='A13' AND status='available'
    DB-->>API: affected_rows=1 (giữ được A13)
    API->>DB: UPDATE SEAT SET status='available', held_by_user_id=NULL WHERE seat_id='A12' AND held_by_user_id=U
    DB-->>API: affected_rows=1 (nhả A12 thành công)
    API->>DB: COMMIT
    DB-->>API: Transaction thành công
    API-->>U: Đã đổi sang ghế A13, ghế A12 đã được nhả

    Note over DB: Kịch bản khác: nếu A13 đã bị người khác giữ trước đó (affected_rows=0 ở bước giữ A13)
    API->>DB: ROLLBACK toàn bộ transaction, không nhả A12
    API-->>U: Ghế A13 đã được giữ bởi người khác, bạn vẫn đang giữ ghế A12 như cũ
```
