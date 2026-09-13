# Sequence Diagram — Enhance: Fare Adjustment After Dispute

Đây là **enhance**, flow mới xử lý tranh chấp giá giữa khách và tài xế được giải quyết *sau khi* cuốc đã kết thúc và tiền đã chia (theo [4-enhance-complete-trip-payment-sqd.md](4-enhance-complete-trip-payment-sqd.md)). Flow này thể hiện yêu cầu điều chỉnh đúng cho cả ba bên (khách, tài xế, hoa hồng) mà không làm sai lệch báo cáo của các cuốc khác trong cùng kỳ.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Ops->>Trip: Resolve fare dispute, set corrected final_amount
    Trip->>Trip: Create FARE_ADJUSTMENT (fare_id, amount_diff, reason, approved_by)

    alt corrected amount lower than original
        Trip->>Payment: Refund rider the difference
        Trip->>Wallet: Debit driver, and reverse commission proportionally
    else corrected amount higher than original
        Trip->>Payment: Charge rider the additional difference
        Trip->>Wallet: Credit driver, and collect additional commission proportionally
    end

    Wallet->>Wallet: Create WALLET_ENTRY (adjustment_id, entry_type=fare_adjustment)
    Wallet-->>Trip: Driver wallet updated

    Trip-->>Ops: Adjustment applied, this trip only
    Note over Trip: Only this TRIP's FARE and WALLET_ENTRY are touched
    Note over Trip: other trips in the same commission period stay untouched
```
