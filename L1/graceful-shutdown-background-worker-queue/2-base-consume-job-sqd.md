# Base sequence — Consume job (ack sớm, không chống mất dữ liệu khi crash)

Đây là **base**, flow "Worker lấy job từ queue và xử lý" ở trạng thái hiện tại — worker ack message ngay sau khi nhận (trước khi xử lý xong) để tránh bị nhận trùng, và không có cơ chế nào xử lý riêng khi có tín hiệu shutdown giữa lúc đang chạy job dài. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu của đề bài xoay quanh việc sửa đúng thời điểm ack và cách worker phản ứng khi bị tắt.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W as Worker
    participant Store as Result Store

    Queue->>W: Deliver message (job=export_report_123)
    W->>Queue: ACK ngay lập tức (để tránh bị deliver lại)
    Note over Queue,W: Từ giờ queue coi job này đã xử lý xong, không giữ lại bản sao nào nữa
    W->>W: Bắt đầu xử lý export file (job mất 3 phút)

    Note over W: Giữa chừng, deploy mới trigger, hạ tầng kill cứng process worker (không có xử lý shutdown riêng)
    W--xW: Process bị kill, file export mới ghi được 40%

    Note over Queue,Store: Message đã bị ACK từ đầu nên queue không còn bản sao để giao lại cho worker khác, job coi như mất, file export dở dang không ai biết để dọn hoặc làm lại
```
