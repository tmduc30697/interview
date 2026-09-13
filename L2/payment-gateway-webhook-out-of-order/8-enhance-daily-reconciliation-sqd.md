# Enhance sequence — Job đối soát hàng ngày với báo cáo cổng thanh toán

Đây là **enhance**, flow hoàn toàn mới phát sinh từ enhance — job chạy hàng ngày so sánh danh sách giao dịch thành công ghi nhận nội bộ (`PAYMENT_TRANSACTION`) với báo cáo giao dịch thành công từ cổng thanh toán, phát hiện và cảnh báo các giao dịch chỉ có ở 1 bên (lệch do webhook bị mất hoàn toàn kể cả sau khi có polling dự phòng, hoặc do lỗi khác).

```mermaid
sequenceDiagram
    participant Scheduler as Reconciliation Job (hàng ngày)
    participant Gateway as Cổng thanh toán
    participant DB as PAYMENT_TRANSACTION store
    participant Discrepancy as RECONCILIATION_DISCREPANCY store
    participant Ops as Đội vận hành

    Scheduler->>Gateway: Tải báo cáo giao dịch thành công trong ngày (theo gateway_transaction_id)
    Gateway-->>Scheduler: Danh sách giao dịch thành công phía cổng thanh toán
    Scheduler->>DB: Đọc danh sách PAYMENT_TRANSACTION status=success ghi nhận nội bộ trong ngày

    Scheduler->>Scheduler: So sánh 2 danh sách theo gateway_transaction_id

    alt Có giao dịch chỉ tồn tại ở báo cáo cổng thanh toán, không có nội bộ
        Scheduler->>Discrepancy: Ghi RECONCILIATION_DISCREPANCY(type=missing_internally)
    else Có giao dịch chỉ tồn tại nội bộ, không có ở báo cáo cổng thanh toán
        Scheduler->>Discrepancy: Ghi RECONCILIATION_DISCREPANCY(type=missing_at_gateway)
    end

    Discrepancy-->>Ops: Gửi cảnh báo danh sách giao dịch lệch trong ngày
    Ops->>Ops: Điều tra nguyên nhân lệch (webhook mất hẳn, lỗi ghi nhận, gian lận...)

    Note over Scheduler,DB: Job chỉ phát hiện và cảnh báo, không tự động sửa trạng thái giao dịch, tránh che giấu nguyên nhân gốc
```
