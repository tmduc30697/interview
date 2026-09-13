# Sequence - Enhance: Chọn ghế A12, giữ chỗ atomic

Đây là flow **enhance hoàn toàn mới**, minh hoạ đúng kịch bản nêu trong đề bài: 2 khách cùng bấm chọn ghế A12 gần như đồng thời (chênh nhau vài chục mili giây). Dùng update nguyên tử có điều kiện `UPDATE seats SET status='held' WHERE seat_id=? AND status='available'`, chỉ 1 request thành công; request thua nhận phản hồi ngay "ghế đã được giữ bởi người khác" và được gợi ý chọn ghế khác, không phải chờ timeout. Đáp ứng **yêu cầu 1** của đề bài.

```mermaid
sequenceDiagram
    participant A as Khách A
    participant B as Khách B
    participant API as Ticketing API
    participant DB as Database

    Note over DB: SEAT A12: status=available
    par 2 request cách nhau vài chục mili giây
        A->>API: Chọn ghế A12
        API->>DB: UPDATE SEAT SET status='held', held_by_user_id=A, held_until=now+5' WHERE seat_id='A12' AND status='available'
    and
        B->>API: Chọn ghế A12
        API->>DB: UPDATE SEAT SET status='held', held_by_user_id=B, held_until=now+5' WHERE seat_id='A12' AND status='available'
    end
    Note over DB: DB serialize 2 UPDATE trên cùng 1 row, chỉ 1 câu thấy status='available' và chuyển được, câu còn lại không khớp điều kiện WHERE
    DB-->>API: affected_rows=1 cho khách A, affected_rows=0 cho khách B
    API-->>A: Đã giữ ghế A12 trong 5 phút, mời điền thông tin thanh toán
    API-->>B: Ghế A12 đã được giữ bởi người khác, gợi ý chọn ghế khác (phản hồi ngay, không timeout)
```
