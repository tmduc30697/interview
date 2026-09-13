# Sequence Diagram — Base: Change Plan With Proration

Đây là **base**, flow "đổi gói giữa chu kỳ" — cũng là tiền đề cho đối soát vì đề bài mô tả đây là nguồn phát sinh "nhiều invoice điều chỉnh nhỏ trong cùng khoảng thời gian ngắn" mà đối soát phải khớp tổng lại với đúng một lượt thu tiền từ processor.

```mermaid
sequenceDiagram
    actor Customer
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)

    Customer->>Billing: Request plan change (new_plan_id)
    Billing->>Billing: Compute unused portion of old plan (credit)
    Billing->>Billing: Create INVOICE (type=proration_credit, negative amount)
    Billing->>Billing: Compute prorated cost of new plan
    Billing->>Billing: Create INVOICE (type=proration_charge, positive amount)

    Billing->>Processor: Charge customer net amount (sum of proration invoices)
    Processor-->>Billing: Charge result (processor_reference_id, status)
    Billing->>Billing: Create PROCESSOR_CHARGE, linked conceptually to both proration invoices

    Billing->>Billing: Update SUBSCRIPTION.plan_id
    Billing-->>Customer: Plan changed, prorated amount charged
```
