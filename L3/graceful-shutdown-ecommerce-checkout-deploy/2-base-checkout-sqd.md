# Base sequence — Checkout (kill cứng khi deploy, lock treo, đơn nửa vời)

Đây là **base**, flow "Checkout: trừ tồn kho, tạo order, gọi cổng thanh toán" ở trạng thái hiện tại — deploy đơn giản kill cứng instance ngay khi có bản mới, không có bước báo readiness=false, không có grace period, không có xử lý khi đang giữ lock. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng khoảng hở này.

```mermaid
sequenceDiagram
    actor Customer
    participant LB as Load Balancer
    participant App as Checkout Instance A
    participant DB as PRODUCT/ORDER store
    participant Gateway as Payment Gateway

    Customer->>LB: Gửi request checkout
    LB->>App: Route request (App vẫn đang nhận traffic bình thường)
    App->>DB: Acquire DISTRIBUTED_LOCK(product_id)
    App->>DB: UPDATE PRODUCT SET stock = stock - 1
    App->>DB: INSERT ORDER (status=pending)

    Note over App: Đúng lúc này, hệ thống deploy trigger kill cứng process ngay lập tức, không có bước drain nào
    App--xApp: Process bị chấm dứt giữa lúc đang gọi Payment Gateway

    Note over DB,Gateway: Tồn kho đã bị trừ, ORDER đã tạo ở status=pending, nhưng không rõ Gateway đã charge hay chưa, và DISTRIBUTED_LOCK không được release, cứ treo cho tới khi tự hết TTL
    Customer->>LB: Không nhận được response nào, client timeout im lặng, không biết đơn có thành công hay không
```
