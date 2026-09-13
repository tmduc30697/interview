# Sequence Diagram — Enhance: Handle Partner Reversal

Đây là **enhance**, flow mới xử lý trường hợp đối tác báo hoàn/đảo một giao dịch (ví dụ tranh chấp thẻ) sau một khoảng thời gian, kể cả khi giao dịch gốc đã đối soát khớp từ trước ở flow [4-enhance-daily-reconciliation-sqd.md](4-enhance-daily-reconciliation-sqd.md). Flow này tái sử dụng cơ chế điều chỉnh có audit trail giống [5-enhance-handle-discrepancy-sqd.md](5-enhance-handle-discrepancy-sqd.md), nhưng khởi phát từ thông báo của đối tác thay vì từ chạy đối soát, và số dư phải được điều chỉnh đúng tại thời điểm phát hiện chứ không lùi lại ngày giao dịch gốc.

```mermaid
sequenceDiagram
    participant Partner as Card Network (Partner)
    participant Recon as Reconciliation Service
    participant Ledger as Ledger Service
    participant Account as Account Service
    participant Audit as Audit Service
    actor Ops as Ops Team

    Partner->>Recon: Send reversal/chargeback notice (partner_transaction_id, reason, reversal_amount)
    Recon->>Ledger: Look up original TRANSACTION by partner_transaction_id
    Ledger-->>Recon: Original transaction found, was already reconciled and matched

    Recon->>Recon: Create PARTNER_REVERSAL_NOTICE (linked to original_transaction_id)
    Recon->>Recon: Create ADJUSTMENT_TRANSACTION (reversal_id, amount=reversal_amount, reason)

    Recon->>Ledger: Post new LEDGER_ENTRY for the reversal, dated at detection time
    Ledger->>Account: Update balance (balance_after) as of now, original entry stays untouched
    Account-->>Ledger: Balance updated

    Recon->>Audit: Log reversal adjustment (before_state, after_state, actor=system, timestamp)
    Audit-->>Recon: Audit entry recorded

    Recon->>Recon: Mark PARTNER_REVERSAL_NOTICE as processed
    Recon-->>Ops: Notify reversal processed, customer balance adjusted
```
