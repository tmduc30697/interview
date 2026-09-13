# Base sequence — Nạp tiền vào ví (đọc-tính-ghi không atomic)

Đây là **base**, flow "Nạp tiền từ ngân hàng liên kết" ở trạng thái hiện tại — service đọc số dư hiện tại, cộng thêm tiền nạp trong bộ nhớ rồi ghi đè lại, không kiểm tra idempotency theo mã giao dịch ngân hàng. Flow này liên quan mật thiết tới enhance vì đây chính là luồng tạo ra khoảng hở race khi callback nạp tiền chạm cùng lúc với giao dịch trừ tiền khác.

```mermaid
sequenceDiagram
    actor Bank as Ngân hàng liên kết
    participant App as Wallet Service
    participant DB as WALLET store

    Bank->>App: Callback báo nạp tiền thành công (bank_ref, amount=50.000đ)
    App->>DB: Đọc balance hiện tại (100.000đ)
    Note over App: Tính balance mới = 100.000 + 50.000 = 150.000 trong bộ nhớ ứng dụng
    App->>DB: Ghi đè balance = 150.000đ
    App->>DB: Ghi TOPUP_TRANSACTION(status=success)
    App-->>Bank: Xác nhận đã xử lý callback

    Note over Bank,DB: Nếu callback bị gửi trùng do retry, bước đọc-tính-ghi này sẽ chạy lại và cộng tiền lần nữa
    Note over App,DB: Nếu đúng lúc này có 1 giao dịch thanh toán khác cũng đang đọc-tính-ghi trên cùng balance, 1 trong 2 lần ghi sẽ bị mất do ghi đè
```
