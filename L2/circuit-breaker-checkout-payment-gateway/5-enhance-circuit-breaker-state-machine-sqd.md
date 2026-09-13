# Enhance sequence — Circuit breaker state machine

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm breaker). Đáp ứng yêu cầu 1 và 4 của đề bài: ngưỡng mở rõ ràng, hành vi fail-fast khi mở, và trạng thái half-open thử lại một lượng nhỏ request trước khi đóng hẳn.

```mermaid
sequenceDiagram
    participant Breaker as CIRCUIT_BREAKER_STATE
    participant Gateway as Payment Gateway
    actor Request as Request thanh toán bất kỳ

    Note over Breaker: state=closed (bình thường)
    loop Theo dõi liên tục
        Request->>Gateway: Gọi thật (breaker closed cho qua)
        Gateway-->>Breaker: Cập nhật tỉ lệ lỗi/latency trong window gần nhất
    end

    alt Tỉ lệ lỗi >50% trong 10 giây gần nhất, hoặc latency p95 vượt ngưỡng
        Breaker->>Breaker: Chuyển state=open, ghi opened_at=now
        Note over Breaker: Toàn bộ request tiếp theo bị fail-fast ngay, không gọi Gateway nữa
    end

    Note over Breaker: Sau khoảng thời gian mở cố định
    Breaker->>Breaker: Chuyển state=half_open
    loop Với 1 lượng nhỏ request thử nghiệm
        Request->>Gateway: Cho phép gọi thật (số lượng giới hạn)
        Gateway-->>Breaker: Kết quả request thử
    end
    alt Toàn bộ request thử nghiệm thành công
        Breaker->>Breaker: Chuyển state=closed, reset bộ đếm lỗi
    else Có request thử nghiệm thất bại
        Breaker->>Breaker: Quay lại state=open, reset lại thời gian mở
    end
```
