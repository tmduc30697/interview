# Sequence Diagram — Enhance: Handle Post-Reconciliation Refund/Chargeback

Đây là **enhance**, flow mới xử lý refund/chargeback xảy ra *sau khi* một đơn hàng đã đối soát khớp thành công (xem [4-enhance-periodic-reconciliation-sqd.md](4-enhance-periodic-reconciliation-sqd.md)). Flow này đảm bảo yêu cầu "không để đơn hàng hiển thị đã đối soát khớp khi thực tế tiền đã được hoàn" — bằng cách mở lại DISCREPANCY cho RECONCILIATION_RESULT liên quan thay vì để trạng thái matched cũ đứng yên.

```mermaid
sequenceDiagram
    participant Gateway as Payment Gateway (Partner)
    participant Recon as Reconciliation Service
    participant Shop as E-commerce Service
    participant Review as Manual Review Queue

    Gateway-->>Recon: Webhook, refund or chargeback event (gateway_reference_code, amount)
    Recon->>Shop: Look up PAYMENT_TRANSACTION and matched ORDER
    Shop-->>Recon: Transaction found, order was previously reconciled as matched

    Recon->>Recon: Create REFUND_EVENT (linked to transaction_id)
    Recon->>Shop: Update ORDER.payment_status (refunded/charged_back)

    Recon->>Recon: Re-open DISCREPANCY on the related RECONCILIATION_RESULT (type=status_mismatch)
    Recon->>Review: Create MANUAL_REVIEW_ITEM if refund needs manual confirmation

    Recon-->>Gateway: Acknowledge webhook received
```
