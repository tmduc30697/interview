# Enhance sequence — Health check probe vẫn được trả lời trong lúc drain

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 4** của đề bài: trong lúc drain, instance không nhận thêm request mới (health check báo not-ready) nhưng vẫn phải trả lời được health check probe, để hạ tầng (Kubernetes/orchestrator) không kill instance ngay lập tức trước khi drain xong.

```mermaid
sequenceDiagram
    participant Infra as Orchestrator (vd Kubernetes)
    participant LB as Load Balancer
    participant App as Checkout Instance A (đang drain)

    Note over App: Nhận SIGTERM, chuyển status=draining, readiness=false
    loop Mỗi vài giây trong suốt grace period
        Infra->>App: GET /healthz/liveness
        App-->>Infra: 200 OK (process vẫn sống, đang xử lý nốt request dở, chưa nên bị kill)
        LB->>App: GET /healthz/readiness
        App-->>LB: 503 Not Ready (readiness=false, ngừng nhận request mới)
        LB->>LB: Không route request mới tới App, nhưng vẫn giữ App trong danh sách theo dõi
    end

    Note over App: App tiếp tục xử lý các request đang chạy dở song song với việc trả lời liveness probe đều đặn
    App->>App: Xử lý xong toàn bộ request đang chạy dở (hoặc hết grace period, xem flow grace-period-timeout-incomplete-order)
    App->>Infra: Chủ động thoát tiến trình (exit code 0)
    Infra->>Infra: Nhận diện process tự thoát sạch, không cần gửi SIGKILL cưỡng chế
```
