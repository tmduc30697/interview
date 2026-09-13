# Enhance sequence — Hết grace period, bị kill cứng, message tự động quay lại queue

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng phần còn lại của **yêu cầu 2**: nếu worker bị kill cứng giữa lúc xử lý (hết grace period mà job chưa xong), message chưa được ack phải tự động được đưa trở lại queue nhờ `visibility_timeout_at` hết hạn, để worker khác xử lý lại, không được để mất.

```mermaid
sequenceDiagram
    actor Queue as Message Queue
    participant W1 as Worker A (đang xử lý)
    participant Orchestrator as Deploy Orchestrator
    participant W2 as Worker B (worker khác trong cụm)

    Queue->>W1: Deliver message (job=bulk_email_88, visibility_timeout=5 phút)
    W1->>W1: Bắt đầu xử lý (nhận tín hiệu shutdown, đang trong grace period)
    Note over W1: Job vẫn chưa xong khi hết grace_period_seconds đã định nghĩa

    Orchestrator->>W1: Hết grace period, kill cứng process
    W1--xW1: Process bị chấm dứt, message chưa được ACK

    Note over Queue: visibility_timeout_at của message này tới hạn (worker A không còn heartbeat/extend), queue tự động coi message là chưa xử lý xong
    Queue->>Queue: Đưa message trở lại trạng thái pending, sẵn sàng giao lại

    Queue->>W2: Deliver lại message (job=bulk_email_88)
    W2->>W2: Xử lý lại job từ đầu (hoặc từ checkpoint nếu có, xem flow checkpoint-idempotent-job)
```
