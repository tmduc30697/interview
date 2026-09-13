# Enhance sequence — Checkout (thứ tự lock cố định toàn cục: order → inventory tăng dần → coupon)

Đây là **enhance** của flow `checkout` đã có ở base, gộp cả 2 kịch bản base (deadlock chéo sản phẩm và deadlock chéo bảng coupon/inventory). So với base, mọi transaction checkout — dù có dùng coupon hay không — đều tuân theo đúng 1 thứ tự lock cố định: order row trước, rồi inventory rows theo `product_id` tăng dần (bỏ qua `added_sequence`), rồi coupon row lock sau cùng. Dùng isolation `READ COMMITTED` kèm `SELECT ... FOR UPDATE` tường minh theo đúng thứ tự này. Đáp ứng yêu cầu 1, 2 và 4 của đề bài.

```mermaid
sequenceDiagram
    actor CustA as Khách A (giỏ: X rồi Y, có coupon)
    actor CustB as Khách B (giỏ: Y rồi X, không coupon)
    participant DB as Database (isolation=READ COMMITTED)
    participant OrdRow as ORDER row
    participant InvX as INVENTORY (product X)
    participant InvY as INVENTORY (product Y)
    participant Cpn as COUPON row

    CustA->>DB: BEGIN Order A checkout
    CustA->>DB: Chuẩn hóa lock_order_product_ids = sort([X, Y]) = [X, Y]
    CustB->>DB: BEGIN Order B checkout
    CustB->>DB: Chuẩn hóa lock_order_product_ids = sort([Y, X]) = [X, Y]

    DB->>OrdRow: Order A: INSERT/lock order row
    DB->>InvX: Order A: SELECT ... FOR UPDATE inventory X (theo thứ tự cố định)
    InvX-->>DB: Lock granted cho Order A

    DB->>OrdRow: Order B: INSERT/lock order row
    DB->>InvX: Order B: SELECT ... FOR UPDATE inventory X - chờ lock
    Note over InvX: Order B luôn lock X trước Y (giống Order A), dù giỏ hàng của khách B thêm Y trước

    DB->>InvY: Order A: SELECT ... FOR UPDATE inventory Y (theo thứ tự cố định)
    InvY-->>DB: Lock granted cho Order A
    DB->>Cpn: Order A: SELECT ... FOR UPDATE coupon row (lock cuối cùng, sau inventory)
    Cpn-->>DB: Lock granted cho Order A
    DB->>DB: Order A: trừ tồn kho X, Y, trừ remaining_uses coupon, COMMIT
    DB->>InvX: Giải phóng lock inventory X

    DB->>InvX: Order B: nhận lock inventory X (vừa giải phóng)
    InvX-->>DB: Lock granted cho Order B
    DB->>InvY: Order B: SELECT ... FOR UPDATE inventory Y (theo thứ tự cố định)
    InvY-->>DB: Lock granted cho Order B
    Note over Cpn: Order B không dùng coupon nên bỏ qua bước lock coupon, nhưng thứ tự order->inventory vẫn nhất quán
    DB->>DB: Order B: trừ tồn kho X, Y, COMMIT

    Note over DB,Cpn: Vì cả 2 nhánh (có/không coupon) đều tuân theo cùng 1 thứ tự lock toàn cục, không còn vòng chờ chéo giữa các order cạnh tranh
```
