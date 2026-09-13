# Enhance sequence — Outlier detection

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base chỉ có health check nhị phân, không phát hiện được instance chậm. Đáp ứng **yêu cầu 1** (phát hiện instance "chậm" dù vẫn healthy theo health check thông thường, giảm trọng số route tới instance đó).

```mermaid
sequenceDiagram
    participant InstanceA as Instance A (latency tăng cao)
    participant HealthCheck as Health Check định kỳ
    participant OutlierDetector as Outlier Detector
    participant LB as Load Balancer

    HealthCheck->>InstanceA: Ping kiểm tra sống/chết
    InstanceA-->>HealthCheck: OK (health check pass bình thường)

    loop Liên tục đo theo từng request thật
        InstanceA->>OutlierDetector: Ghi OUTLIER_SAMPLE(latency_ms của response vừa xử lý)
    end
    OutlierDetector->>OutlierDetector: Tính latency_p95_ms của InstanceA trong cửa sổ gần nhất
    alt latency_p95_ms vượt ngưỡng so với các instance khác trong cụm
        OutlierDetector->>LB: Yêu cầu giảm routing_weight của InstanceA
        LB->>LB: Giảm dần tỉ lệ request route tới InstanceA, dù health check vẫn pass
        Note over LB,InstanceA: Response time tệ được phát hiện và xử lý sớm, không chờ tới khi health check phát hiện lỗi hệ thống
    else latency bình thường
        OutlierDetector->>LB: Giữ nguyên routing_weight
    end
```
