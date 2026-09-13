# Enhance sequence — Checkout (readiness=false ngay lập tức, hoàn tất giao dịch đang chạy, release lock)

Đây là **enhance** của flow `checkout` đã có ở base, đáp ứng **yêu cầu 1 và yêu cầu 3** của đề bài. So với base: khi nhận SIGTERM, instance báo `readiness=false` ngay lập tức để load balancer ngừng gửi request mới, nhưng vẫn tiếp tục xử lý request đang chạy cho tới khi xong; sau khi giao dịch hoàn tất, `DISTRIBUTED_LOCK` được release tường minh thay vì để treo.

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer
    participant App as Checkout Instance A
    participant DB as PRODUCT/ORDER/LOCK store
    participant Gateway as Payment Gateway

    Customer->>LB: Gửi request checkout (request X)
    LB->>App: Route request X (App vẫn ready)
    App->>DB: Acquire DISTRIBUTED_LOCK(product_id)
    App->>DB: UPDATE PRODUCT SET stock=stock-1, INSERT ORDER(status=inventory_reserved)

    Note over App: Nhận SIGTERM ngay lúc này, giữa lúc request X đang chạy
    App->>LB: Báo readiness=false ngay lập tức (không đợi request X xong)
    LB->>LB: Ngừng route request mới tới App, các request đang chạy dở (như X) không bị huỷ

    App->>Gateway: Tiếp tục gọi charge cho request X (không bị cắt ngang)
    Gateway-->>App: Charge thành công
    App->>DB: UPDATE ORDER SET status=completed
    App->>DB: Release DISTRIBUTED_LOCK(product_id), released_by=explicit
    App-->>Customer: 200 OK, đơn hàng hoàn tất

    App->>App: Không còn request nào đang xử lý dở, instance tắt hẳn (status=stopped)
```
