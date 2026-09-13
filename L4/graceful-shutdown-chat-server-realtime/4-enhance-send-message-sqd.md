# Enhance sequence — Send message (chuyển giao hàng đợi bền vững, không gửi trùng khi reconnect)

Đây là **enhance** của flow `send-message` đã có ở base, đáp ứng **yêu cầu 1 và yêu cầu 3** của đề bài. So với base: tin nhắn chưa được ACK tại thời điểm shutdown được ghi vào store bền vững (`status=queued_durable`) thay vì chỉ giữ trong memory, để instance khác tiếp tục gửi. Song song đó, mỗi tin nhắn có `delivered_at` được ghi lại ngay khi gửi thành công, nên nếu server bị ngắt giữa lúc gửi ACK, instance mới khi reconnect biết tin nhắn đã delivered và không gửi lại trùng.

```mermaid
sequenceDiagram
    actor Sender
    participant I1 as Chat Server Instance A (đang drain)
    participant Store as Durable Message Store
    actor Receiver
    participant I2 as Chat Server Instance B

    Sender->>I1: Gửi tin nhắn M1
    I1->>Store: INSERT MESSAGE(status=pending)
    I1->>Receiver: Đẩy M1 qua WebSocket
    I1->>Store: UPDATE MESSAGE SET status=delivered, delivered_at=now (ghi ngay khi gửi thành công, trước khi có ACK)

    Note over I1: Instance A nhận tín hiệu shutdown ngay lúc này, chưa kịp nhận ACK từ Receiver cho M1
    I1->>Store: Đảm bảo mọi MESSAGE chưa có acked_by_client_at đều đã ở status=delivered hoặc queued_durable (không còn nằm riêng trong memory)
    I1->>I1: Đóng kết nối tới Receiver (xem chi tiết ở flow drain-disconnect-close-code)

    Receiver->>I2: Reconnect vào Instance B
    I2->>Store: SELECT MESSAGE WHERE conversation_id=... AND acked_by_client_at IS NULL
    Store-->>I2: M1 (status=delivered, delivered_at đã có, acked_by_client_at=null)
    I2->>I2: Vì M1 đã delivered_at trước đó, không gửi lại M1, chỉ chờ ACK
    Receiver->>I2: Gửi ACK cho M1 (client vốn đã nhận M1 từ trước khi mất kết nối)
    I2->>Store: UPDATE MESSAGE SET status=acked, acked_by_client_at=now
```
