# Base sequence — Deploy (rolling, chưa có blue-green)

Đây là **base**, flow "Deploy phiên bản mới" ở trạng thái hiện tại — rolling deploy trực tiếp, không có bản song song để test trước. Flow này liên quan mật thiết tới enhance vì toàn bộ yêu cầu 1-3 của đề bài đều nhằm cải tổ lại đúng flow này.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Old as Deployment cũ (đang chạy)
    participant DB as Shared Database
    participant Router as Router/Load Balancer
    actor Client

    CI->>DB: Áp dụng DB_MIGRATION mới trực tiếp lên schema hiện tại
    Note over DB: Không đảm bảo backward-compatible — chỉ cần bản mới chạy đúng
    CI->>Old: Dừng từng instance cũ, khởi động instance phiên bản mới (rolling restart)
    Old-->>Client: Các CLIENT_CONNECTION (WebSocket/request dài) đang mở trên instance bị dừng → bị cắt ngang
    CI->>Router: ROUTER_CONFIG.target_deployment_id trỏ theo instance mới khi rolling xong
    Router-->>Client: Traffic mới đi vào phiên bản mới
    Note over CI,Router: Không có bước smoke test tự động, nếu có lỗi phải chờ dashboard/user report mới phát hiện
```
