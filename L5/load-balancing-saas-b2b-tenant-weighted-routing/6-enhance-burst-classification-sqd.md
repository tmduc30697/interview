# Enhance sequence — Phân loại traffic spike hợp lệ vs bất thường

Đây là **enhance**, flow hoàn toàn mới xử lý khi 1 tenant vượt ngưỡng `max_rps` bình thường. Đáp ứng yêu cầu 2 của đề bài: phân biệt spike hợp lệ (tenant đang chạy chiến dịch riêng của họ) với traffic bất thường/lỗi client, để không chặn nhầm nhu cầu chính đáng nhưng vẫn bảo vệ tenant khác.

```mermaid
sequenceDiagram
    actor Tenant as Tenant đang có traffic tăng đột biến
    participant GW as Gateway Instance
    participant SharedCounter as RATE_LIMIT_COUNTER store
    participant Classifier as Traffic Spike Classifier
    participant SpikeLog as TRAFFIC_SPIKE_EVENT

    Tenant->>GW: Traffic tăng 50x so với baseline
    GW->>SharedCounter: current_rps vượt max_rps thông thường
    GW->>Classifier: Yêu cầu phân loại spike (tenant_id, spike_ratio=50x)
    Classifier->>Classifier: Kiểm tra tín hiệu, error_rate của tenant vẫn thấp, pattern request ổn định, không có dấu hiệu retry storm/lỗi client
    Classifier-->>GW: classification = legit_spike
    GW->>SpikeLog: Ghi TRAFFIC_SPIKE_EVENT (classification=legit_spike, action_taken=allow_with_burst)
    GW->>GW: Cho phép request đi tiếp trong hạn mức burst_multiplier (thay vì chặn cứng ở max_rps)
    GW-->>Tenant: Request được xử lý bình thường trong giới hạn burst tạm thời

    Note over Classifier,SpikeLog: Nếu ngược lại phát hiện error_rate cao bất thường hoặc pattern bất thường (vd retry storm), classification = abnormal, action_taken = throttle/block để bảo vệ tenant khác
```
