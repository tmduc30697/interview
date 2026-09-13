# Enhance sequence — Circuit breaker phối hợp với LB

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chỉ dựa vào health check thủ công để loại instance lỗi. Đáp ứng **yêu cầu 5** (circuit breaker phối hợp với LB: instance liên tục trả lỗi 5xx tự động bị loại khỏi vòng quay trong một khoảng thời gian rồi thử lại dần qua half-open, không chờ health check thủ công).

```mermaid
sequenceDiagram
    actor Customers
    participant LB as Load Balancer
    participant InstanceB as Instance B (bắt đầu lỗi 5xx liên tục)
    participant Breaker as Circuit Breaker (theo từng instance)

    loop Một loạt request liên tiếp
        Customers->>LB: Request
        LB->>InstanceB: Forward request (circuit_state=closed)
        InstanceB-->>LB: Trả 5xx
        LB->>Breaker: Ghi nhận lỗi, tăng consecutive_5xx_count
    end

    alt consecutive_5xx_count vượt ngưỡng
        Breaker->>Breaker: Chuyển circuit_state=open, ghi CIRCUIT_BREAKER_EVENT(from=closed, to=open)
        Breaker->>LB: Yêu cầu loại InstanceB khỏi vòng quay ngay lập tức
        LB-->>Customers: Không route request nào tới InstanceB nữa trong thời gian breaker mở
    end

    Note over Breaker: Sau một khoảng thời gian cố định
    Breaker->>Breaker: Chuyển circuit_state=half_open, ghi CIRCUIT_BREAKER_EVENT(from=open, to=half_open)
    LB->>InstanceB: Cho phép một lượng nhỏ request thử nghiệm đi qua
    InstanceB-->>LB: Kết quả request thử

    alt Request thử nghiệm thành công
        Breaker->>Breaker: Chuyển circuit_state=closed, reset consecutive_5xx_count
        LB->>InstanceB: Route lại bình thường
    else Vẫn còn lỗi 5xx
        Breaker->>Breaker: Quay lại circuit_state=open, gia hạn thời gian mở
    end
```
