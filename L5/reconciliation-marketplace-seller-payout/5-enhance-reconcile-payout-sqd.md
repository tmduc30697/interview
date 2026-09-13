# Sequence Diagram — Enhance: Reconcile Payout

Đây là **enhance**, flow mới hoàn toàn: đối soát giữa `PAYOUT.total_amount` (số tiền sàn tính) và trạng thái thực tế của `PAYOUT_TRANSFER`. Flow này thể hiện yêu cầu phân biệt "đã lên lệnh chuyển" với "seller thực nhận" — chỉ coi kỳ đã đóng khi transfer được đối tác xác nhận thành công — và tạo `PAYOUT_HOLD` cho seller có sai lệch mà không ảnh hưởng các seller khác.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Payout as Payout Service
    participant Gateway as Payment Gateway (Partner)
    participant Hold as Payout Hold Service

    Recon->>Payout: List PAYOUT_TRANSFER for the period, status=pending
    Recon->>Gateway: Query actual transfer status by gateway_reference_code

    loop for each PAYOUT_TRANSFER
        Gateway-->>Recon: Transfer status (confirmed/failed/returned) and actual amount
        alt confirmed and amount matches
            Recon->>Payout: Mark PAYOUT_TRANSFER confirmed_at
            Recon->>Recon: Mark PAYOUT as reconciled
        else failed or returned, or amount mismatch
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (payout_id, amount_diff, type)
            Recon->>Hold: Create PAYOUT_HOLD for this seller's next PAYOUT
            Note over Hold: Only this seller's next payout is held
            Note over Hold: other sellers in the same run are unaffected
        end
    end

    Recon-->>Ops: Reconciliation summary, discrepancies and holds created
```
