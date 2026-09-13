# Sequence Diagram — Enhance: Handle Refund Clawback

Đây là **enhance**, flow mới xử lý trường hợp một `ORDER` đã được tính vào `PAYOUT` đã chuyển tiền, nhưng sau đó phát sinh hoàn tiền cho người mua. Flow này thể hiện yêu cầu "phát hiện được các khoản âm" thay vì chỉ cộng dồn một chiều, bằng cách tạo `PAYOUT_CLAWBACK` để trừ vào kỳ payout tiếp theo của đúng seller đó.

```mermaid
sequenceDiagram
    participant Buyer
    participant Order as Order Service
    participant Payout as Payout Service

    Buyer->>Order: Request refund for a completed order
    Order->>Order: Process refund, create REFUND_EVENT
    Order-->>Payout: Notify refund on ORDER already included in a past PAYOUT

    Payout->>Payout: Look up ORDER.payout_id, confirm it was already paid out
    Payout->>Payout: Create PAYOUT_CLAWBACK (refund_event_id, amount, status=pending)

    Note over Payout: Clawback is not deducted retroactively from the past payout
    Note over Payout: it is applied as a negative line item in the seller's next PAYOUT_PERIOD

    Payout-->>Order: Clawback recorded, will offset next payout
```

