# Sequence - Enhance: Sơ đồ ghế cập nhật real-time cho mọi khách đang xem

Đây là flow **enhance** của `view-ticket-availability` (so với base). Khác biệt so với base: thay vì client tự poll 1 con số tổng mỗi vài giây, mỗi thay đổi trạng thái ghế (`available`/`held`/`sold`) được phát ngay qua kênh real-time (WebSocket/SSE) tới mọi client đang xem sơ đồ ghế của cùng sự kiện, với độ trễ chấp nhận được (vd dưới 300ms), tránh hiển thị 1 ghế "còn trống" trong khi thực tế vừa bị giữ vài trăm mili giây trước. Đáp ứng **yêu cầu 5** của đề bài.

```mermaid
sequenceDiagram
    participant C as Khách C (đang xem sơ đồ ghế)
    participant WS as Kênh real-time (WebSocket)
    participant API as Ticketing API
    participant DB as Database
    participant OtherUser as Khách khác

    C->>WS: Kết nối, subscribe cập nhật ghế của sự kiện E
    WS-->>C: Gửi trạng thái hiện tại toàn bộ sơ đồ ghế (snapshot)
    C->>C: Hiển thị sơ đồ, ghế A12 đang "available"

    OtherUser->>API: Giữ ghế A12 thành công
    API->>DB: UPDATE SEAT A12 status='held'
    API->>WS: Phát SEAT_STATUS_EVENT(seat_id=A12, new_status=held)
    Note over WS: Độ trễ phát tới mọi client subscribe dưới 300ms
    WS-->>C: Đẩy cập nhật seat_id=A12, new_status=held
    C->>C: Cập nhật ngay ghế A12 chuyển màu "đã giữ" trên sơ đồ

    Note over C: Khách C không còn thấy A12 "còn trống" trong khi thực tế vừa bị giữ, tránh bấm chọn rồi nhận lỗi
```
