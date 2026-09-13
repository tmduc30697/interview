# Sequence Diagram — Enhance: Daily Reconciliation

Đây là **enhance**, flow chính hoàn toàn mới mà đề bài yêu cầu: đối soát cuối ngày giữa ledger nội bộ và file batch từ đối tác mạng thẻ. Flow này thể hiện yêu cầu chuẩn hóa "ngày giao dịch" giữa hai múi giờ, phân loại 3 loại sai lệch với mức ưu tiên khác nhau, và ngưỡng cảnh báo tự động khi tổng sai lệch vượt mức — tất cả chưa tồn tại ở base.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Partner as Card Network (Partner)
    participant Recon as Reconciliation Service
    participant Ledger as Ledger Service
    participant Alert as Alert Service

    Partner->>Recon: Upload settlement batch file (business_date, partner_timezone)
    Note over Recon: File may arrive late, or after internal day-close cutoff

    Recon->>Recon: Normalize each record's transaction_datetime to internal standard business_date
    Recon->>Recon: Create RECONCILIATION_RUN for business_date

    loop for each PARTNER_SETTLEMENT_RECORD
        Recon->>Ledger: Look up TRANSACTION by partner_reference_id
        alt matched, amount equal
            Ledger-->>Recon: Transaction found, amount matches
            Recon->>Recon: Mark record as reconciled, no discrepancy
        else exists at partner, missing in ledger
            Ledger-->>Recon: No matching transaction found
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=missing_in_ledger, priority=high)
        else exists in ledger, partner did not confirm
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=missing_in_partner, priority=medium)
        else matched but amount differs
            Ledger-->>Recon: Transaction found, amount mismatch
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=amount_mismatch, priority=high)
        end
    end

    Recon->>Recon: Sum total_discrepancy_amount for the run
    alt total_discrepancy_amount exceeds threshold
        Recon->>Alert: Trigger RECONCILIATION_ALERT (threshold, actual amount)
        Alert-->>Ops: Urgent notification, day-close blocked
        Ops->>Recon: Acknowledge alert, begin triage
    else within threshold
        Recon->>Recon: Mark RECONCILIATION_RUN as completed
        Recon-->>Ops: Reconciliation summary, day-close allowed
    end
```
