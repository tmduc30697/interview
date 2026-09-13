# Sequence Diagram — Enhance: Reconcile Billing Cycle

Đây là **enhance**, flow chính hoàn toàn mới: đối soát giữa nhóm `INVOICE` (kể cả các invoice proration nhỏ phát sinh khi đổi gói, xem [3-base-change-plan-proration-sqd.md](3-base-change-plan-proration-sqd.md)) với đúng một `PROCESSOR_CHARGE`. Flow thể hiện việc tách `processing_fee` khỏi `gross_amount` để so đúng `net_amount`, phân biệt trạng thái "đang chờ retry" (dunning) với "thất bại hẳn", và có bước xử lý riêng cho giao dịch cận ranh giới chu kỳ do lệch múi giờ.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Billing as Billing Service
    participant Processor as Payment Processor (Partner)

    Ops->>Recon: Trigger reconciliation for cycle_period
    Recon->>Billing: Group INVOICE by subscription within cycle boundary (using unified cutoff rule)

    loop for each group of invoices tied to one charge attempt
        Recon->>Billing: Sum group invoice amounts (invoice_total)
        Recon->>Processor: Look up PROCESSOR_CHARGE by processor_reference_id
        Processor-->>Recon: gross_amount, processing_fee, net_amount, status

        alt charge status = succeeded and invoice_total equals net_amount
            Recon->>Recon: Create RECONCILIATION_MATCH (match_status=matched)
        else charge status = retrying (dunning in progress)
            Recon->>Processor: Check DUNNING_ATTEMPT history
            Processor-->>Recon: next_retry_at still in the future
            Recon->>Recon: Mark group as pending, exclude from discrepancy this run
            Note over Recon: Not closed yet, still eligible for a future successful retry
        else charge status = failed permanently
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=charge_failed)
        else invoice_total does not equal net_amount but charge succeeded
            Recon->>Recon: Create RECONCILIATION_DISCREPANCY (type=amount_mismatch)
        else transaction near cycle boundary (ambiguous cutoff)
            Recon->>Recon: Apply boundary rule, re-classify into correct cycle before comparing
        end
    end

    Recon-->>Ops: Reconciliation summary for the cycle
```
