# Sequence Diagram — Enhance: Driver Payout Reconciliation

Đây là **enhance**, flow mới đối soát ở cả mức chi tiết từng cuốc lẫn mức tổng hợp theo đợt thanh toán (`PAYOUT_BATCH`). Flow này thể hiện yêu cầu "đối soát ở cả mức chi tiết và mức tổng hợp để phát hiện sai lệch dù nhỏ không bị pha loãng" khi tài xế có khối lượng cuốc lớn mỗi ngày.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Wallet as Driver Wallet Service
    participant Payout as Payout Batch Service

    Recon->>Payout: List PAYOUT_BATCH for the period
    loop for each PAYOUT_BATCH
        Recon->>Wallet: Sum all WALLET_ENTRY (trip earnings, adjustments, cancellations) for this driver/period
        Wallet-->>Recon: Sum of individual entries

        alt batch total matches sum of entries
            Recon->>Recon: Mark PAYOUT_BATCH as reconciled
        else mismatch found
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (level=batch, batch_id, amount_diff)
        end

        Recon->>Wallet: Drill into individual TRIP-level entries for this batch
        loop for each TRIP in batch
            Recon->>Recon: Verify FARE.final_amount, commission, driver credit are all consistent for this trip
            alt trip-level inconsistency found
                Recon->>Recon: Create RECONCILIATION_DISCREPANCY (level=trip, trip_id, amount_diff)
            end
        end
    end

    Recon-->>Ops: Reconciliation summary, discrepancies at both trip and batch level
```
