# Enhance sequence — Canary monitoring & auto rollback

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base chỉ có dashboard 5xx chung, kiểm tra thủ công). Đáp ứng yêu cầu 2, 4 và 5 của đề bài: metric riêng cho thanh toán (không chỉ 5xx), đo theo từng version gần thời gian thực, và rollback tự động cho ngưỡng nghiêm trọng mà không chờ người trực ca.

```mermaid
sequenceDiagram
    participant Canary as Deployment canary
    participant Stable as Deployment stable
    participant Metric as PAYMENT_METRIC_WINDOW store
    participant Monitor as Auto Rollback Monitor
    participant Policy as ROLLBACK_POLICY store
    participant Router as Router / TRAFFIC_SPLIT

    loop Mỗi window ngắn (vài giây tới 1 phút)
        Canary->>Metric: Ghi payment_failure_rate + http_5xx_rate riêng cho canary
        Stable->>Metric: Ghi payment_failure_rate + http_5xx_rate riêng cho stable (làm baseline)
    end

    Monitor->>Metric: Lấy payment_failure_rate của canary và stable trong window gần nhất
    Monitor->>Policy: Lấy threshold_type=relative_to_baseline, threshold_value=2x, window_minutes=2

    alt payment_failure_rate của canary chưa gấp đôi baseline
        Monitor->>Monitor: Tiếp tục theo dõi, cho phép tiến trình tăng traffic ở flow "Deploy"
    else payment_failure_rate của canary vượt ngưỡng nghiêm trọng (gấp đôi baseline trong 2 phút)
        Monitor->>Router: Kích hoạt rollback ngay, không chờ on-call xác nhận
        Router->>Router: Tạo ROLLBACK_EVENT(triggered_by=auto_policy), canary_percent=0
        Router-->>Stable: Toàn bộ traffic quay lại stable
        Monitor-->>Monitor: Gửi cảnh báo cho on-call để điều tra thêm (thông tin, không phải chờ duyệt)
        Note over Router: Đơn hàng đang dở dang trên canary được xử lý ở flow "In-flight order rollback handling"
    end
```
