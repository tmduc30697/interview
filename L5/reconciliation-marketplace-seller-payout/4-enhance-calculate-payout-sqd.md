# Sequence Diagram — Enhance: Calculate Payout

Đây là **enhance**, flow "tính payout" đã có ở base ([2-base-calculate-payout-sqd.md](2-base-calculate-payout-sqd.md)) nhưng nay thay đổi ở hai điểm: (1) dùng `PAYOUT_PERIOD.cutoff_at` cố định để xác định đơn nào thuộc kỳ nào thay vì mốc thời gian ngầm định, tránh tính trùng/bỏ sót; (2) mỗi `ORDER` dùng đúng `commission_rate_snapshot` đã chốt tại thời điểm hoàn tất thay vì `COMMISSION_CONFIG` hiện hành.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Payout as Payout Service
    participant Order as Order Service
    participant Hold as Payout Hold Service
    participant Gateway as Payment Gateway (Partner)

    Ops->>Payout: Trigger payout run for PAYOUT_PERIOD (cutoff_at)
    Payout->>Payout: Lock cutoff_at, only orders completed_at < cutoff_at are eligible

    Payout->>Hold: Check for active PAYOUT_HOLD on any seller
    Hold-->>Payout: List of held sellers, excluded from this run

    Payout->>Order: List eligible completed ORDER per seller (using commission_rate_snapshot)
    Order-->>Payout: Orders with order_amount, commission_amount already snapshotted

    Payout->>Payout: Sum orders per seller, subtract pending PAYOUT_CLAWBACK if any
    Payout->>Payout: Create PAYOUT (total_amount) per non-held seller

    Payout->>Gateway: Initiate transfer (amount, seller bank info)
    Gateway-->>Payout: Transfer accepted (gateway_reference_code)
    Payout->>Payout: Create PAYOUT_TRANSFER (status=pending)

    Payout-->>Ops: Payout run summary, held sellers listed separately
```
