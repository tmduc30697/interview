# Sequence Diagram — Enhance: Complete Trip and Split Payment

Đây là **enhance**, flow "hoàn tất cuốc và chia tiền" đã có ở base ([2-base-complete-trip-payment-sqd.md](2-base-complete-trip-payment-sqd.md)) nhưng nay thay đổi ở chỗ: cuốc trả tiền mặt không đi qua Payment Service để thu tiền từ khách, nhưng hoa hồng vẫn phải bị trừ vào ví tài xế theo hướng ngược lại, thay vì chỉ cộng dồn một chiều như base.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Driver->>Trip: Mark trip completed (final route, distance, payment_method)
    Trip->>Trip: Compute FARE.final_amount from final route

    alt payment_method = card/wallet
        Trip->>Payment: Charge rider final_amount
        Payment-->>Trip: Payment captured
        Trip->>Wallet: Credit driver (final_amount minus commission)
        Wallet->>Wallet: Create WALLET_ENTRY (entry_type=trip_earning)
    else payment_method = cash
        Note over Rider,Driver: Rider pays driver directly, cash never touches the platform
        Trip->>Wallet: Debit driver for commission only (rider already paid driver in cash)
        Wallet->>Wallet: Create WALLET_ENTRY (entry_type=cash_trip_commission, amount negative)
    end

    Wallet-->>Trip: Driver wallet updated
    Trip-->>Rider: Trip receipt (final fare)
    Trip-->>Driver: Trip earning summary
```
