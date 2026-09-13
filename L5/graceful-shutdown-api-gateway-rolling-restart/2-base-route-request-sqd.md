# Base sequence — Route request (chỉ dựa vào health check định kỳ)

Đây là **base**, flow "Gateway route request tới instance" ở trạng thái hiện tại — pool instance chỉ được cập nhật qua health check polling định kỳ (ví dụ mỗi 5 giây), gateway không có cách nào biết instance sắp tắt trước khi health check tự phát hiện. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm rút ngắn/loại bỏ khoảng trễ phát hiện này và các hệ quả của nó.

```mermaid
sequenceDiagram
    actor Client
    participant GW as API Gateway
    participant HC as Health Checker (polling mỗi 5s)
    participant I1 as Instance A (đang chạy bình thường)

    HC->>I1: GET /health (định kỳ)
    I1-->>HC: 200 OK
    HC->>GW: Cập nhật pool: Instance A = healthy

    Note over I1: Vận hành trigger deploy, gửi SIGTERM cho Instance A ngay bây giờ
    I1->>I1: Bắt đầu shutdown, ngừng xử lý request mới ngầm

    Client->>GW: Gửi request
    GW->>I1: Route request (pool vẫn coi Instance A là healthy vì chưa tới lần health check kế tiếp)
    I1-->>GW: Connection reset / timeout (instance đang tắt)
    GW-->>Client: 502 Bad Gateway

    Note over HC,GW: Chỉ khi lần health check định kỳ kế tiếp chạy tới (có thể trễ vài giây) gateway mới biết Instance A unhealthy và loại khỏi pool
    HC->>I1: GET /health (lần kế tiếp)
    I1-->>HC: Không phản hồi
    HC->>GW: Cập nhật pool: Instance A = unhealthy
```
