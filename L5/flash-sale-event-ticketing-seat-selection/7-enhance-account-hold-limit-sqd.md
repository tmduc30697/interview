# Sequence - Enhance: Giới hạn số ghế giữ đồng thời theo tài khoản (atomic)

Đây là flow **enhance hoàn toàn mới**, mô tả trường hợp khách mở nhiều tab/thiết bị bằng cùng 1 tài khoản để chọn nhiều ghế khác nhau cho cùng sự kiện gần như đồng thời. Mỗi tài khoản chỉ được giữ tối đa `max_hold_per_account` ghế cùng lúc (vd 4 ghế), việc kiểm tra và tăng `active_hold_count` phải atomic để không bị vượt hạn mức khi nhiều request giữ ghế gửi lên gần như đồng thời từ cùng tài khoản. Đáp ứng **yêu cầu 3** của đề bài.

```mermaid
sequenceDiagram
    participant Tab1 as Tab 1 (ghế B5)
    participant Tab2 as Tab 2 (ghế B6)
    participant API as Ticketing API
    participant Counter as ACCOUNT_HOLD_COUNTER
    participant DB as Database

    Note over Counter: user=U, event=E, active_hold_count=3 (đã giữ 3/4 ghế cho phép, còn đúng 1 suất)
    par 2 tab bấm chọn ghế gần như đồng thời
        Tab1->>API: Chọn ghế B5
        API->>Counter: UPDATE active_hold_count = active_hold_count + 1 WHERE user=U AND event=E AND active_hold_count < 4
    and
        Tab2->>API: Chọn ghế B6
        API->>Counter: UPDATE active_hold_count = active_hold_count + 1 WHERE user=U AND event=E AND active_hold_count < 4
    end
    Note over Counter: DB serialize 2 UPDATE trên cùng 1 row counter, chỉ 1 câu thấy active_hold_count=3 < 4 và tăng được lên 4
    Counter-->>API: affected_rows=1 cho Tab 1, affected_rows=0 cho Tab 2
    API->>DB: UPDATE SEAT B5 status='held' (đã qua kiểm tra hạn mức)
    API-->>Tab1: Đã giữ ghế B5
    API-->>Tab2: Bạn đã đạt giới hạn 4 ghế giữ đồng thời, không thể giữ thêm ghế B6
    Note over Counter: Hạn mức theo tài khoản không bị vượt dù 2 request tới gần như cùng lúc từ 2 thiết bị khác nhau
```
