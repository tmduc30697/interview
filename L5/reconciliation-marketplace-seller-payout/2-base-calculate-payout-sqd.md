# Sequence Diagram — Base: Calculate Payout

Đây là **base**, flow "tính và chuyển payout cho seller" theo kỳ — tiền đề bắt buộc cho đối soát: chính flow này tạo ra `PAYOUT` (số tiền sàn tính seller được nhận) và `PAYOUT_TRANSFER` (lệnh chuyển tiền thực), hai con số mà đối soát sau này phải khớp với nhau. Ở base, flow này còn đơn giản, chưa xử lý cutoff nhất quán hay lịch sử phí.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Payout as Payout Service
    participant Order as Order Service
    participant Gateway as Payment Gateway (Partner)

    Ops->>Payout: Trigger payout run for period
    Payout->>Order: List completed ORDER for each seller in period
    Order-->>Payout: Orders with order_amount, commission_amount

    Payout->>Payout: Sum orders, apply current COMMISSION_CONFIG rate
    Payout->>Payout: Create PAYOUT (total_amount) per seller

    Payout->>Gateway: Initiate transfer (amount, seller bank info)
    Gateway-->>Payout: Transfer accepted (gateway_reference_code)
    Payout->>Payout: Create PAYOUT_TRANSFER (status=pending)

    Payout-->>Ops: Payout run summary

    Note over Payout,Gateway: No confirmation yet that seller actually received the money
    Note over Payout,Gateway: and no check against what the transfer partner actually moved
```
