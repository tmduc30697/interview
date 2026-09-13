# Sequence Diagram - Enhance: graceful-shutdown-release

Đây là flow **enhance mới**: khi worker nhận tín hiệu terminate có báo trước từ autoscale (graceful shutdown signal, ví dụ SIGTERM), worker chủ động release ngay lock/job đang giữ thay vì để coordinator chờ tới khi TTL hết hạn, giúp job khả dụng lại nhanh hơn nhiều so với chờ timeout tự nhiên. Đáp ứng yêu cầu 4.

```mermaid
sequenceDiagram
    participant AS as Autoscaler
    participant W as Worker (đang giữ job J1)
    participant Q as Job Queue/DB
    participant W2 as Worker khác

    AS->>W: Gửi graceful shutdown signal (sắp terminate)
    W->>W: Nhận signal, dừng nhận job mới
    W->>Q: Release claim job J1 ngay lập tức (status=AVAILABLE, xoá lease)
    Q-->>W: OK
    W->>AS: Xác nhận sẵn sàng bị terminate
    AS->>W: Terminate worker
    W2->>Q: Poll job status=AVAILABLE
    Q-->>W2: Trả về job J1 (khả dụng ngay, không phải chờ TTL 60s)
    W2->>Q: Claim job J1
    Q-->>W2: OK
```
