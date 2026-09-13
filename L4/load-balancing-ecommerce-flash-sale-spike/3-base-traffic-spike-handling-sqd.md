# Base sequence — Traffic spike handling

Đây là **base**, flow "Xử lý khi traffic tăng đột biến" ở trạng thái hiện tại: khi flash sale khiến traffic tăng 20 lần, LB tiếp tục route mọi request như bình thường, không có outlier detection, không shed load, không circuit breaker — cả cụm bị dồn tải tới khi sập. Flow này là tiền đề cho **yêu cầu 1** (outlier detection), **yêu cầu 2** (shed load có kiểm soát) và **yêu cầu 5** (circuit breaker) vì cả ba đều nhằm xử lý đúng tình huống này.

```mermaid
sequenceDiagram
    actor Customers as Hàng loạt customer (traffic x20)
    participant LB as Load Balancer
    participant InstanceA as Instance A (latency tăng cao nhưng health check vẫn pass)
    participant InstanceB as Instance B (bắt đầu trả 5xx liên tục)
    participant InstanceC as Instance C

    Customers->>LB: Request đồng loạt tăng vọt
    LB->>InstanceA: Health check định kỳ
    InstanceA-->>LB: OK (health check pass dù response time đã tệ)
    LB->>InstanceB: Health check định kỳ
    InstanceB-->>LB: OK (health check pass dù đang trả 5xx cho request thật)
    LB->>LB: Vẫn route đều theo thuật toán hiện tại cho cả InstanceA, InstanceB, InstanceC
    LB-->>Customers: Một phần request rơi vào InstanceA (chậm) hoặc InstanceB (lỗi 5xx)
    Note over LB,InstanceC: Khi cả cụm đạt ngưỡng tải tối đa, không có cơ chế shed load nào, toàn bộ request tiếp tục dồn vào cho tới khi timeout đồng loạt
```
