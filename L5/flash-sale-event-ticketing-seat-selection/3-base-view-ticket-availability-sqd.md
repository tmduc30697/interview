# Sequence - Base: Xem số vé còn lại (poll định kỳ, không real-time)

Đây là flow **base**: khách xem trang sự kiện, giao diện chỉ hiển thị 1 con số tổng "còn X vé", được client tự poll lại mỗi vài giây, không có cơ chế đẩy cập nhật real-time. Vì không có ghế cụ thể, không cần đồng bộ trạng thái từng ghế theo thời gian thực — đây là tiền đề cho yêu cầu 5 của đề bài, khi ghế cụ thể xuất hiện thì việc đồng bộ trạng thái từng ghế real-time trở nên cần thiết.

```mermaid
sequenceDiagram
    participant U as Khách hàng đang xem trang sự kiện
    participant FE as Trang sự kiện
    participant API as Ticketing API
    participant DB as Database

    loop Client tự poll mỗi 5 giây
        FE->>API: GET remaining_tickets
        API->>DB: Đọc remaining_tickets
        DB-->>API: remaining_tickets
        API-->>FE: Trả về số vé còn lại
        FE-->>U: Hiển thị "còn X vé"
    end
    Note over U: Chỉ thấy 1 con số tổng, không có sơ đồ ghế nào để cập nhật theo từng đơn vị
```
