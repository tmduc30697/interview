# Sequence Diagram — Enhance: Periodic Reconciliation

Đây là **enhance**, flow chính hoàn toàn mới mà đề bài yêu cầu: đối soát định kỳ giữa `ORDER` và `PAYMENT_TRANSACTION`. Flow thể hiện việc khớp theo mã tham chiếu duy nhất, phân loại 4 loại sai lệch (thiếu giao dịch, thừa giao dịch, sai số tiền, sai trạng thái), và tính idempotent khi chạy lại nhiều lần trên cùng khoảng thời gian — tất cả chưa tồn tại ở base.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Shop as E-commerce Service
    participant Review as Manual Review Queue

    Ops->>Recon: Trigger reconciliation run (period_start, period_end)
    Recon->>Recon: Check for existing RECONCILIATION_RUN on this period

    alt run already exists and completed
        Recon-->>Ops: Reuse existing RECONCILIATION_RESULT, skip re-processing matched pairs
    else new or incomplete run
        Recon->>Recon: Create/resume RECONCILIATION_RUN

        loop for each ORDER with payment_status=paid in period
            Recon->>Shop: Look up PAYMENT_TRANSACTION by payment_reference_code
            alt matched, amount and status agree
                Shop-->>Recon: Transaction found, amount and status match
                Recon->>Recon: Create RECONCILIATION_RESULT (match_status=matched)
            else order paid but no transaction found
                Shop-->>Recon: No matching transaction
                Recon->>Recon: Create DISCREPANCY (type=missing_transaction)
            else transaction exists but order not marked paid
                Recon->>Recon: Create DISCREPANCY (type=missing_order_update)
            else amount differs
                Recon->>Recon: Create DISCREPANCY (type=amount_mismatch)
            else status differs
                Recon->>Recon: Create DISCREPANCY (type=status_mismatch)
            end
        end

        Recon->>Recon: Mark RECONCILIATION_RUN as completed
    end

    loop for each unresolved DISCREPANCY needing human judgment
        Recon->>Review: Create MANUAL_REVIEW_ITEM (status=open)
    end

    Recon-->>Ops: Reconciliation summary, discrepancies by type
```
