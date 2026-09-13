# Sequence Diagram — Base: Checkout Payment

Đây là **base**, flow "thanh toán khi checkout" — tiền đề bắt buộc cho đối soát: chính flow này tạo ra `ORDER` với `payment_reference_code` và `PAYMENT_TRANSACTION` từ webhook của cổng thanh toán, hai dữ liệu mà job đối soát sau này phải khớp với nhau.

```mermaid
sequenceDiagram
    actor Customer
    participant Shop as E-commerce Service
    participant Gateway as Payment Gateway (Partner)

    Customer->>Shop: Checkout, confirm order
    Shop->>Shop: Create ORDER (status=pending_payment, generate payment_reference_code)
    Shop->>Gateway: Create charge request (payment_reference_code, amount)
    Gateway-->>Shop: Redirect customer to gateway payment page
    Customer->>Gateway: Enter payment info, confirm
    Gateway->>Gateway: Process payment

    Gateway-->>Shop: Webhook callback (gateway_reference_code, status, amount)
    Shop->>Shop: Create PAYMENT_TRANSACTION from webhook payload
    Shop->>Shop: Update ORDER.payment_status based on callback
    Shop-->>Customer: Show order confirmation

    Note over Shop,Gateway: Order's local payment_status and gateway's actual record can drift apart
    Note over Shop,Gateway: if webhook is lost, delayed, or delivered twice
```
