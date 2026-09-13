# Enhance sequence — Ngắt kết nối để drain kèm close code riêng biệt

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 2** của đề bài: khi ngắt kết nối WebSocket để drain, server gửi kèm close code riêng cho "server draining" (ví dụ 4000), khác với close code lỗi bất thường (ví dụ 1006), để client phân biệt và tự reconnect ngay thay vì hiển thị lỗi gây hoang mang.

```mermaid
sequenceDiagram
    actor Client
    participant I1 as Chat Server Instance A (đang drain)
    participant I2 as Chat Server Instance B

    Note over I1: Instance A nhận tín hiệu shutdown, chuyển CONNECTION.status=draining
    I1->>Client: WebSocket close(code=4000, reason="server_draining")
    Client->>Client: Nhận diện code=4000 khác với code=1006 (lỗi bất thường)

    alt code=4000 server_draining
        Client->>Client: Không hiển thị lỗi cho người dùng, tự động reconnect ngay lập tức
        Client->>I2: Mở kết nối WebSocket mới
        I2-->>Client: Kết nối thành công, tiếp tục phiên chat bình thường
    else code=1006 hoặc các mã lỗi bất thường khác
        Client->>Client: Hiển thị trạng thái "mất kết nối", reconnect theo backoff tăng dần
    end
```
