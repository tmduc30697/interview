# Sequence Diagram — Enhance: Handle Post-Reconciliation Refund

Đây là **enhance**, flow mới xử lý trường hợp khách hủy gói giữa chu kỳ và được hoàn lại phần tiền chưa dùng *sau khi* invoice gốc đã đối soát khớp (xem [5-enhance-reconcile-billing-cycle-sqd.md](5-enhance-reconcile-billing-cycle-sqd.md)). Flow này đảm bảo yêu cầu "không để hệ thống báo cáo doanh thu vẫn tính khoản đã hoàn là doanh thu thực" bằng cách mở lại đối soát cho match liên quan.

```mermaid
sequenceDiagram
    actor Customer
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)
    participant Recon as Reconciliation Service

    Customer->>Billing: Cancel subscription mid-cycle
    Billing->>Billing: Compute unused amount to refund
    Billing->>Processor: Issue refund for unused amount

    Processor-->>Billing: Refund confirmed
    Billing->>Billing: Create REFUND_EVENT (invoice_id, amount)

    Billing-->>Recon: Notify refund on an already-reconciled INVOICE
    Recon->>Recon: Look up RECONCILIATION_MATCH containing this invoice
    Recon->>Recon: Re-open RECONCILIATION_DISCREPANCY on that match (type=refunded_after_match)

    Recon-->>Billing: Revenue report excludes refunded amount going forward
```
