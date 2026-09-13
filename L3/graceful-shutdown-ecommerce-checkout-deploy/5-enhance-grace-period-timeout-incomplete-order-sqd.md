# Enhance sequence — Hết grace period giữa lúc giao dịch chưa xong, ghi state rõ ràng thay vì kill cứng

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 2 và phần còn lại của yêu cầu 3** của đề bài: định nghĩa grace period tối đa (30 giây) — nếu giao dịch checkout chưa xong khi hết grace period, hệ thống ghi lại state rõ ràng (`failed_incomplete` kèm lý do) thay vì kill cứng để lại đơn nửa vời, đồng thời đảm bảo `DISTRIBUTED_LOCK` được release tường minh trước khi tắt hẳn.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Checkout Instance A (đang drain, grace_period=30s)
    participant DB as PRODUCT/ORDER/LOCK store
    participant Gateway as Payment Gateway (đang phản hồi chậm)

    Note over App: Nhận SIGTERM lúc t=0, readiness=false, shutdown_deadline_at=t+30s
    Customer->>App: Request checkout Y đang xử lý dở từ trước SIGTERM
    App->>DB: Acquire DISTRIBUTED_LOCK(product_id), UPDATE stock, INSERT ORDER(status=inventory_reserved)
    App->>Gateway: Gọi charge cho request Y
    Note over Gateway: Gateway phản hồi rất chậm, không trả lời kịp

    Note over App: t=30s, hết grace period, request Y vẫn chưa có kết quả từ Gateway
    App->>DB: UPDATE ORDER SET status=failed_incomplete, incomplete_reason="grace_period_expired_during_payment_call"
    App->>DB: Release DISTRIBUTED_LOCK(product_id), released_by=explicit (không để treo cho tới khi tự hết TTL)
    App-->>Customer: 503 kèm order_id, giải thích giao dịch chưa xác nhận được, cần kiểm tra lại trạng thái đơn sau
    App->>App: Tắt hẳn (status=stopped)

    Note over DB: Vì stock đã bị trừ nhưng ORDER=failed_incomplete được ghi rõ, job đối soát nền có thể phát hiện và tự động hoàn lại tồn kho nếu Gateway xác nhận sau đó là chưa charge thành công
```
