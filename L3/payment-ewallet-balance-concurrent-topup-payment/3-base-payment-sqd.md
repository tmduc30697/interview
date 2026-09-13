# Base sequence — Thanh toán bằng số dư ví (đọc-tính-ghi không atomic)

Đây là **base**, flow "Thanh toán tại merchant" ở trạng thái hiện tại — service đọc số dư, kiểm tra đủ hay không trong bộ nhớ ứng dụng, rồi mới trừ tiền và ghi lại, không dựa trên update nguyên tử tại tầng lưu trữ. Flow này liên quan mật thiết tới enhance vì đây chính là luồng có thể duyệt nhầm thanh toán sai hoặc gây số dư âm khi 2 giao dịch chạm cùng 1 số dư gần như đồng thời.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Wallet Service
    participant DB as WALLET store
    participant Merchant

    Customer->>App: Quét mã QR thanh toán 120.000đ tại merchant
    App->>DB: Đọc balance hiện tại (100.000đ)
    Note over App: Kiểm tra trong bộ nhớ ứng dụng, 100.000 < 120.000 nên từ chối
    App-->>Customer: Từ chối thanh toán, "số dư không đủ"

    Note over App,DB: Nếu đúng lúc callback nạp 50.000đ vừa cộng xong ở tầng lưu trữ nhưng chưa kịp phản ánh vào lần đọc balance ở trên, thực tế số dư đã là 150.000đ đủ trả nhưng vẫn bị từ chối sai
    Note over Customer,DB: Ngược lại, nếu 2 lệnh thanh toán khác nhau cùng đọc balance 100.000đ rồi cùng tính đủ và cùng ghi trừ, cả 2 có thể cùng thành công gây số dư âm
```
