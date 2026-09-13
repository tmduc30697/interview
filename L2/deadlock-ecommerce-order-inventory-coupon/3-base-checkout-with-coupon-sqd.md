# Base sequence — Checkout với/không coupon (thứ tự lock khác nhau giữa 2 nhánh)

Đây là **base**, flow checkout khi 2 nhánh code (đơn dùng coupon và đơn không dùng coupon) lock tài nguyên theo thứ tự khác nhau: nhánh dùng coupon validate/lock coupon trước rồi mới trừ tồn kho, nhánh không dùng coupon trừ tồn kho trước rồi mới ghi order. Đây là kịch bản rủi ro cross-table nêu ở yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor CustA as Khách A (đơn dùng coupon, có sản phẩm Z)
    actor CustB as Khách B (đơn không dùng coupon, có sản phẩm Z)
    participant DB as Database
    participant Cpn as COUPON row
    participant InvZ as INVENTORY (product Z)

    CustA->>DB: BEGIN Order A (có coupon)
    CustB->>DB: BEGIN Order B (không coupon)

    DB->>Cpn: Order A: SELECT ... FOR UPDATE coupon row (validate trước)
    Cpn-->>DB: Lock granted cho Order A

    DB->>InvZ: Order B: SELECT ... FOR UPDATE inventory Z (trừ tồn kho trước, chưa cần coupon)
    InvZ-->>DB: Lock granted cho Order B

    DB->>InvZ: Order A: SELECT ... FOR UPDATE inventory Z (trừ tồn kho sau khi coupon OK) - chờ lock
    Note over InvZ: Inventory Z đang bị Order B giữ lock

    DB->>Cpn: Order B: (giả sử đơn B sau đó cũng cần chạm coupon table để ghi thống kê) SELECT ... FOR UPDATE coupon row - chờ lock
    Note over Cpn: Coupon row đang bị Order A giữ lock

    Note over DB,InvZ: Order A chờ Order B (inventory Z), Order B chờ Order A (coupon), DEADLOCK chéo giữa 2 bảng khác nhau
    DB-->>CustA: Lỗi deadlock, transaction rollback
```
