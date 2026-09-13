# Sequence Diagram — Enhance: Handle Discrepancy

Đây là **enhance**, flow mới xử lý một `RECONCILIATION_DISCREPANCY` được tạo ra từ flow đối soát ([4-enhance-daily-reconciliation-sqd.md](4-enhance-daily-reconciliation-sqd.md)). Flow này thể hiện yêu cầu "không thể sửa trực tiếp số dư mà không qua bước ghi nhận giao dịch điều chỉnh tương ứng" và audit trail đầy đủ — số dư khách hàng (vốn chỉ được cập nhật qua LEDGER_ENTRY ở base) nay chỉ có thể thay đổi thêm qua một ADJUSTMENT_TRANSACTION được audit.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Recon as Reconciliation Service
    participant Ledger as Ledger Service
    participant Account as Account Service
    participant Audit as Audit Service

    Ops->>Recon: Review open discrepancies, ordered by priority
    Recon-->>Ops: Discrepancy detail (type, amount_diff, related transaction/record)
    Ops->>Recon: Approve adjustment with reason

    Recon->>Recon: Create ADJUSTMENT_TRANSACTION (discrepancy_id, amount, reason, created_by)
    Recon->>Ledger: Post new LEDGER_ENTRY for the adjustment
    Ledger->>Account: Update balance (balance_after)
    Account-->>Ledger: Balance updated
    Ledger-->>Recon: Ledger entry posted

    Recon->>Audit: Log adjustment (before_state, after_state, actor, timestamp)
    Audit-->>Recon: Audit entry recorded

    Recon->>Recon: Mark RECONCILIATION_DISCREPANCY as resolved
    Recon-->>Ops: Discrepancy resolved, balance corrected
```
