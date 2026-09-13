# Sequence Diagram — Enhance: Rollback via Feature Flag

Đây là **enhance**, flow hoàn toàn mới: khi hệ thống đã chuyển đọc số dư sang `balance_cents` nhưng phát hiện lỗi nghiêm trọng, ops phải có cách quay lại đọc `balance_float` ngay lập tức bằng feature flag, không cần deploy lại code.

```mermaid
sequenceDiagram
    actor Ops as Ops Team
    participant Flag as Read Source Flag Service
    participant App as Expense App Service
    actor User

    Note over App: Đang đọc số dư từ balance_cents (trạng thái sau cutover)

    Ops->>Flag: Phát hiện lỗi nghiêm trọng, set read_source = amount_float
    Flag-->>Ops: Flag updated, propagated ngay lập tức

    User->>App: Xem số dư tài khoản
    App->>Flag: Check current read_source flag
    Flag-->>App: read_source = amount_float
    App->>App: Đọc balance_float thay vì balance_cents
    App-->>User: Hiển thị số dư từ cột float, không cần deploy lại code

    Note over Ops,Flag: Sau khi khắc phục xong lỗi ở cột cents, ops set lại flag để chuyển đọc sang cents
```
