# Sequence Diagram — Base: Charge Subscription Cycle

Đây là **base**, flow "thu tiền định kỳ hàng tháng" — tiền đề bắt buộc cho đối soát: chính flow này tạo ra `INVOICE` (type=recurring) và `PROCESSOR_CHARGE` tương ứng, hai dữ liệu mà đối soát sau này phải khớp với nhau theo từng chu kỳ.

```mermaid
sequenceDiagram
    participant Scheduler as Billing Scheduler
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)

    Scheduler->>Billing: Trigger billing for subscriptions due today
    Billing->>Billing: Create INVOICE (type=recurring, amount=plan.monthly_price)
    Billing->>Processor: Charge customer (invoice_id, amount)

    Processor-->>Billing: Charge result (processor_reference_id, status)
    Billing->>Billing: Create PROCESSOR_CHARGE from result
    Billing->>Billing: Update INVOICE.status based on charge result

    Billing-->>Scheduler: Cycle billing completed

    Note over Billing,Processor: No systematic check yet that invoice totals for the period
    Note over Billing,Processor: actually match what the processor collected
```
