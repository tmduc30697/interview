# Enhance sequence — Chuyển giao/kết thúc rõ ràng typing indicator và cuộc gọi khi drain

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 4** của đề bài: khi server drain lúc user đang gõ hoặc đang trong cuộc gọi voice/video, các `PRESENCE_STATE` này phải được chuyển giao hoặc kết thúc rõ ràng, không để lại trạng thái "đang gõ" treo vĩnh viễn ở phía người nhận.

```mermaid
sequenceDiagram
    actor UserA as User A (đang gõ)
    participant I1 as Chat Server Instance A (đang drain)
    participant Store as PRESENCE_STATE store
    actor UserB as User B (đang xem "A đang gõ...")
    participant I2 as Chat Server Instance B

    UserA->>I1: Bắt đầu gõ trong conversation C
    I1->>Store: INSERT PRESENCE_STATE(type=typing, expires_at=now+10s)
    I1->>UserB: Đẩy sự kiện "A đang gõ..."

    Note over I1: Instance A nhận tín hiệu shutdown giữa lúc User A vẫn đang gõ
    I1->>Store: UPDATE PRESENCE_STATE SET expires_at=now (hết hạn ngay, không chờ tới +10s)
    I1->>UserB: Đẩy sự kiện "A đã ngừng gõ" (kết thúc rõ ràng thay vì treo)
    I1->>UserA: Đóng kết nối (close_code=4000 server_draining)
    UserA->>I2: Reconnect, tiếp tục gõ nếu vẫn đang gõ
    I2->>Store: INSERT PRESENCE_STATE mới (type=typing) nếu UserA còn đang gõ

    Note over I1,Store: Với cuộc gọi voice/video, tương tự nhưng KHÔNG tự kết thúc cuộc gọi
    Note over I1,Store: PRESENCE_STATE(type=in_call) được đánh dấu status=transferring, tín hiệu signaling (SDP/ICE) được chuyển tiếp sang Instance B để cuộc gọi tiếp tục không gián đoạn, chỉ kết thúc hẳn nếu instance mới không nhận chuyển giao kịp trong thời gian giới hạn
```
