# Enhance sequence — Nạp tiền atomic + idempotent theo mã giao dịch ngân hàng

Đây là **enhance**, flow "Nạp tiền từ ngân hàng liên kết" sau khi áp update nguyên tử có điều kiện. So với base, flow này thay đổi ở 2 điểm: (1) kiểm tra `bank_reference_id` đã tồn tại trong `TOPUP_TRANSACTION` chưa trước khi cộng tiền, để callback trùng do retry không cộng 2 lần, (2) việc cộng số dư dùng update nguyên tử `balance = balance + amount` ngay tại tầng lưu trữ, không có khoảng hở giữa lúc callback bắt đầu xử lý và lúc số dư thực sự được cộng, nên các giao dịch trừ tiền chạy song song luôn thấy đúng 1 trong 2 trạng thái trước/sau, không thấy trạng thái lỡ dở.

```mermaid
sequenceDiagram
    actor Bank as Ngân hàng liên kết
    participant App as Wallet Service
    participant DB as WALLET + TOPUP_TRANSACTION store

    Bank->>App: Callback báo nạp tiền thành công (bank_ref=TX-001, amount=50.000đ)
    App->>DB: Kiểm tra TOPUP_TRANSACTION theo bank_reference_id=TX-001

    alt bank_reference_id đã tồn tại (callback trùng)
        DB-->>App: Đã có TOPUP_TRANSACTION(status=success)
        App-->>Bank: Xác nhận đã xử lý (idempotent, không cộng tiền lần 2)
    else bank_reference_id chưa tồn tại
        App->>DB: UPDATE WALLET SET balance = balance + 50000, ghi TOPUP_TRANSACTION(bank_ref=TX-001, status=success) trong 1 giao dịch atomic
        DB-->>App: Atomic update thành công, ghi WALLET_LEDGER_ENTRY(delta=+50000)
        App-->>Bank: Xác nhận đã xử lý
    end

    Note over App,DB: Vì update atomic ngay tại tầng lưu trữ, các lệnh trừ tiền chạy song song không thể đọc thấy trạng thái balance nửa vời giữa lúc cộng
```
