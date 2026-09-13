# Enhance sequence — 2 lệnh thanh toán đồng thời chỉ 1 lệnh thành công

Đây là **enhance**, flow mới phát sinh từ enhance — khách quét mã QR thanh toán ở 2 merchant gần như đồng thời từ 2 thiết bị/session trong khi số dư chỉ đủ cho 1 trong 2 lệnh. Flow này minh hoạ trực tiếp tác dụng của update nguyên tử có điều kiện ở file `6-enhance-payment-sqd.md`: 2 update chạm cùng 1 dòng `WALLET` được tầng lưu trữ tuần tự hoá, chỉ update nào còn thấy đủ số dư mới khớp điều kiện và thành công.

```mermaid
sequenceDiagram
    actor Customer
    participant App1 as Wallet Service (session 1)
    participant App2 as Wallet Service (session 2)
    participant DB as WALLET store

    Note over Customer,App2: Số dư hiện tại 30.000đ, khách quét QR thanh toán 30.000đ tại merchant A và 25.000đ tại merchant B gần như cùng lúc

    Customer->>App1: Thanh toán 30.000đ tại merchant A
    Customer->>App2: Thanh toán 25.000đ tại merchant B

    par 2 update nguyên tử chạm cùng 1 dòng WALLET
        App1->>DB: UPDATE WALLET SET balance = balance - 30000 WHERE id=wallet_id AND balance >= 30000
    and
        App2->>DB: UPDATE WALLET SET balance = balance - 25000 WHERE id=wallet_id AND balance >= 25000
    end

    Note over DB: Tầng lưu trữ tuần tự hoá 2 update trên cùng dòng, update nào chạy trước thấy balance=30.000 đủ điều kiện, khớp và trừ trước

    DB-->>App1: Update khớp 1 dòng, balance mới = 0
    App1-->>Customer: "Thanh toán tại merchant A thành công"

    DB-->>App2: Update chạy sau, balance lúc này = 0, không khớp điều kiện balance >= 25000
    App2-->>Customer: Từ chối ngay, "số dư không đủ" tại merchant B

    Note over DB: Không có thời điểm nào cả 2 update cùng đọc thấy balance=30.000 rồi cùng trừ thành công, nên số dư không thể bị âm
```
