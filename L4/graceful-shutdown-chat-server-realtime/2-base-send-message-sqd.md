# Base sequence — Send message (chỉ giữ trong memory, mất khi instance tắt)

Đây là **base**, flow "Gửi tin nhắn qua WebSocket" ở trạng thái hiện tại — tin nhắn đang chờ gửi/chờ ACK chỉ tồn tại trong bộ nhớ của instance đang giữ kết nối, và khi instance tắt (không phân biệt lý do), kết nối bị đóng đột ngột không kèm mã lý do rõ ràng. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu của đề bài đều nhằm vá đúng những lỗ hổng này.

```mermaid
sequenceDiagram
    actor Sender
    participant I1 as Chat Server Instance A
    actor Receiver

    Sender->>I1: Gửi tin nhắn M1 (qua WebSocket)
    I1->>I1: Lưu M1 vào hàng đợi gửi trong memory (status=pending)
    I1->>Receiver: Đẩy M1 qua WebSocket
    Note over I1: Đang chờ Receiver gửi ACK xác nhận đã nhận M1

    Note over I1: Vận hành trigger deploy/maintenance, tắt Instance A ngay bây giờ, không phân biệt shutdown chủ động hay crash
    I1--xI1: Process tắt, hàng đợi gửi trong memory (bao gồm M1 nếu chưa kịp nhận ACK) biến mất hoàn toàn

    Receiver->>Receiver: Kết nối WebSocket bị đóng đột ngột không kèm mã lý do, client hiển thị lỗi kết nối chung chung
    Note over Receiver: Người dùng không biết đây là do server bảo trì hay lỗi thật, tự đoán và tự reconnect sau vài giây với backoff ngẫu nhiên
```
