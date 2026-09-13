# Enhance sequence — Deploy (canary theo request, tăng dần)

Đây là **enhance**, cùng flow "Deploy" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 1 của đề bài: canary chỉ nhận dưới 1% traffic ban đầu, chọn ngẫu nhiên theo từng request chứ không cố định theo user, và chỉ tăng dần khi các bước trước đó ổn định.

```mermaid
sequenceDiagram
    participant CI as CI/CD
    participant Stable as Deployment stable (đang chạy)
    participant Canary as Deployment canary (mới)
    participant Split as TRAFFIC_SPLIT store
    participant Router as Router
    actor Customer

    CI->>Canary: Deploy bản mới thành canary, song song với Stable
    CI->>Split: Tạo TRAFFIC_SPLIT(canary_percent=0.5%, selection_strategy=per_request)
    Customer->>Router: Gửi request checkout
    Router->>Split: Random theo từng request (không theo user cố định)
    alt Rơi vào nhóm canary (dưới 1%)
        Router->>Canary: Route request này sang canary
    else Rơi vào nhóm stable (đa số)
        Router->>Stable: Route request này sang stable
    end
    Note over Router,Customer: Cùng 1 khách hàng có thể request lần này vào canary, lần sau vào stable — tránh 1 khách bị dính lỗi canary liên tục nhiều lần
    Note over Split: Sau khi vượt qua bước giám sát (xem flow "Canary monitoring & auto rollback"), canary_percent tăng dần theo bậc (0.5% → 5% → 25% → 100%), mỗi bậc đều được giám sát lại trước khi tăng tiếp
```
