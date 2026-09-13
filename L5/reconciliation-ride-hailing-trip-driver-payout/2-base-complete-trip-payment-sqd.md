# Sequence Diagram — Base: Complete Trip and Split Payment

Đây là **base**, flow "hoàn tất cuốc xe và chia tiền" — tiền đề bắt buộc cho đối soát: chính flow này chốt `FARE.final_amount`, tính hoa hồng theo `COMMISSION_CONFIG`, và ghi `WALLET_ENTRY` cho tài xế. Ba con số (khách trả, tài xế nhận, hoa hồng giữ lại) sinh ra ở đây chính là ba khoản mà đối soát sau này phải khớp với nhau cho từng cuốc.

```mermaid
sequenceDiagram
    actor Rider
    actor Driver
    participant Trip as Trip Service
    participant Payment as Payment Service
    participant Wallet as Driver Wallet Service

    Driver->>Trip: Mark trip completed (final route, distance)
    Trip->>Trip: Compute FARE.final_amount from final route
    Trip->>Payment: Charge rider final_amount

    Payment-->>Trip: Payment captured
    Trip->>Trip: Apply COMMISSION_CONFIG rate to final_amount

    Trip->>Wallet: Credit driver (final_amount minus commission)
    Wallet->>Wallet: Create WALLET_ENTRY (amount, entry_type=trip_earning)
    Wallet-->>Trip: Driver wallet updated

    Trip-->>Rider: Trip receipt (final fare)
    Trip-->>Driver: Trip earning summary
```
