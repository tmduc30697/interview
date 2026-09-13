# Enhance sequence — Thanh toán bằng update nguyên tử có điều kiện

Đây là **enhance**, flow "Thanh toán tại merchant" sau khi áp update nguyên tử có điều kiện. So với base, flow này không còn đọc balance rồi tính đủ/thiếu trong bộ nhớ ứng dụng nữa — việc trừ tiền và kiểm tra đủ số dư gộp làm 1 câu update nguyên tử duy nhất tại tầng lưu trữ (`balance = balance - amount WHERE balance >= amount`), nên callback nạp tiền chạm đúng lúc thanh toán vẫn cho kết quả đúng theo thứ tự thực sự xử lý tại tầng lưu trữ, không theo thời điểm khách bấm nút. Nếu tại đúng thời điểm ghi số dư không đủ, hệ thống từ chối ngay, không giữ pending chờ nạp xong rồi âm thầm thử lại.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Wallet Service
    participant DB as WALLET + PAYMENT_TRANSACTION store
    participant Merchant

    Customer->>App: Quét mã QR thanh toán 120.000đ tại merchant
    App->>DB: UPDATE WALLET SET balance = balance - 120000 WHERE id=wallet_id AND balance >= 120000

    alt Update khớp 1 dòng (số dư đủ tại đúng thời điểm ghi)
        DB-->>App: Atomic update thành công, balance mới = 150.000 - 120.000 = 30.000
        App->>DB: Ghi PAYMENT_TRANSACTION(status=success), WALLET_LEDGER_ENTRY(delta=-120000)
        App-->>Merchant: Xác nhận thanh toán thành công
        App-->>Customer: "Thanh toán thành công"
    else Update không khớp dòng nào (số dư không đủ tại đúng thời điểm ghi)
        DB-->>App: 0 dòng bị ảnh hưởng
        App->>DB: Ghi PAYMENT_TRANSACTION(status=rejected_insufficient_balance)
        App-->>Customer: Từ chối ngay, "số dư không đủ", không giữ pending chờ nạp xong
    end

    Note over App,DB: Nếu callback nạp 50.000đ được xử lý atomic trước lệnh trừ này, số dư 150.000 đủ trả nên nhánh thành công sẽ xảy ra dù khách bấm thanh toán trước khi thấy tiền nạp về
```
