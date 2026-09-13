# Enhance sequence — Consume job (ngừng prefetch ngay, ack sau khi xong và ghi bền vững)

Đây là **enhance** của flow `consume-job` đã có ở base, đáp ứng **yêu cầu 1 và 2** của đề bài. So với base: khi nhận tín hiệu shutdown, worker ngừng ngay việc poll/prefetch message mới nhưng tiếp tục hoàn tất job đang xử lý dở, và chỉ ACK sau khi job đã hoàn tất toàn bộ và kết quả đã ghi bền vững — không còn ack ngay sau khi nhận message như base.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W as Worker
    participant Store as Result Store

    Queue->>W: Deliver message (job=export_report_123, visibility_timeout=5 phút)
    W->>W: Bắt đầu xử lý export file (không ACK vội)

    Note over W: Giữa lúc đang xử lý, nhận tín hiệu shutdown (SIGTERM)
    W->>Queue: Ngừng poll/prefetch message mới ngay lập tức
    W->>W: Tiếp tục xử lý job đang dở tới khi hoàn tất (không nhận thêm job mới)

    W->>Store: Ghi file export hoàn chỉnh (bền vững)
    Store-->>W: OK
    W->>Queue: ACK job=export_report_123 (chỉ ack sau khi ghi xong)
    Queue-->>W: Xác nhận xoá message khỏi queue

    W->>W: Không còn job nào đang xử lý dở, worker tắt hẳn
```
