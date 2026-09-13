# Sequence Diagram — Enhance: Manual Review Queue

Đây là **enhance**, flow mới cho đội vận hành xử lý các `MANUAL_REVIEW_ITEM` phát sinh từ đối soát ([4-enhance-periodic-reconciliation-sqd.md](4-enhance-periodic-reconciliation-sqd.md), [5-enhance-handle-post-reconciliation-refund-sqd.md](5-enhance-handle-post-reconciliation-refund-sqd.md)). Flow này đảm bảo yêu cầu "không có sai lệch nào bị bỏ quên" bằng cách theo dõi trạng thái xử lý tường minh cho từng sai lệch cần con người can thiệp.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Review as Manual Review Queue
    participant Recon as Reconciliation Service
    participant Shop as E-commerce Service

    Ops->>Review: List open MANUAL_REVIEW_ITEM, ordered by discrepancy type
    Review-->>Ops: Item detail (discrepancy type, amount_diff, related order/transaction)

    Ops->>Review: Claim item, mark status=in_progress
    Ops->>Review: Investigate, decide resolution (adjust order, contact gateway support, ignore as noise)
    Ops->>Recon: Submit resolution

    Recon->>Shop: Apply correction if needed (update order status/amount)
    Recon->>Recon: Mark related DISCREPANCY as resolved
    Recon->>Review: Mark MANUAL_REVIEW_ITEM as resolved (resolved_at)

    Recon-->>Ops: Confirmation, item closed
```
