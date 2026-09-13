# Sequence Diagram — Enhance: Cancel Trip With Partial Fee

Đây là **enhance**, flow mới xử lý cuốc bị hủy giữa chừng sau khi tài xế đã di chuyển một đoạn. Flow này thể hiện yêu cầu "phân biệt rõ luồng tiền của cuốc hoàn tất và cuốc hủy có phí bồi thường" — không tính hoa hồng theo logic cuốc bình thường, mà tạo `TRIP_CANCELLATION` với phí hủy một phần riêng.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Rider->>Trip: Cancel trip while driver is en route
    Trip->>Trip: Compute partial cancellation_fee based on distance driven

    Trip->>Trip: Create TRIP_CANCELLATION (cancelled_by=rider, cancellation_fee)
    Trip->>Payment: Charge rider cancellation_fee only (not full fare)
    Payment-->>Trip: Payment captured

    Trip->>Wallet: Credit driver full cancellation_fee, no commission deducted
    Wallet->>Wallet: Create WALLET_ENTRY (entry_type=cancellation_compensation)
    Wallet-->>Trip: Driver wallet updated

    Trip->>Trip: Mark TRIP status=cancelled
    Trip-->>Rider: Cancellation confirmed, fee charged
    Trip-->>Driver: Compensation credited
```
