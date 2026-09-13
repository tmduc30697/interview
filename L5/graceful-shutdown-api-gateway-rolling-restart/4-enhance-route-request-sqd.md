# Enhance sequence — Route request (drain signal chủ động, không tái dùng keep-alive cũ)

Đây là **enhance** của flow `route-request` đã có ở base, đáp ứng **yêu cầu 1 và 2** của đề bài. So với base: instance chủ động gửi `DRAIN_SIGNAL` cho gateway ngay khi nhận SIGTERM (không đợi health check polling phát hiện), gateway loại instance khỏi pool routing ngay lập tức. Với các kết nối HTTP keep-alive đang mở tới instance đó, gateway đóng kết nối (hoặc đánh dấu closing) để request tiếp theo trên client bắt buộc mở kết nối mới tới instance khác, không bao giờ tái sử dụng kết nối cũ tới instance đang chết.

```mermaid
sequenceDiagram
    actor Client
    participant GW as API Gateway
    participant I1 as Instance A (sắp shutdown)
    participant I2 as Instance B (còn khỏe)

    Client->>GW: Request 1 (mở keep-alive connection C1)
    GW->>I1: Route qua C1
    I1-->>GW: 200 OK
    GW-->>Client: 200 OK (C1 vẫn giữ mở)

    Note over I1: Vận hành gửi SIGTERM cho Instance A
    I1->>GW: DRAIN_SIGNAL(instance=A, reason=sigterm) -- chủ động, không đợi health check
    GW->>GW: Đánh dấu Instance A = draining, loại khỏi pool routing ngay
    GW->>I1: Đóng kết nối keep-alive C1 (status=closing)

    Client->>GW: Request 2 trên C1 (client tưởng vẫn dùng lại kết nối cũ)
    GW-->>Client: Báo C1 đã đóng, buộc mở kết nối mới
    Client->>GW: Request 2 (mở keep-alive connection C2 mới)
    GW->>I2: Route qua C2 (Instance A đã bị loại khỏi pool)
    I2-->>GW: 200 OK
    GW-->>Client: 200 OK

    I1->>I1: Hoàn tất drain các request đang xử lý dở, sau đó tắt hẳn
```
