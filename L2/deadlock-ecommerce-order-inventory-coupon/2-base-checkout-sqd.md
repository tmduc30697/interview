# Base sequence — Checkout (trừ tồn kho theo thứ tự giỏ hàng, gây deadlock chéo sản phẩm)

Đây là **base**, flow checkout ở trạng thái hiện tại: code trừ tồn kho theo đúng `added_sequence` (thứ tự khách thêm vào giỏ), không sort lại theo `product_id`. Đây chính là kịch bản deadlock kinh điển nêu ở yêu cầu 1 của đề bài — đơn hàng A thêm X trước Y, đơn hàng B thêm Y trước X, chạy đồng thời.

```mermaid
sequenceDiagram
    actor CustA as Khách A (giỏ hàng: X rồi Y)
    actor CustB as Khách B (giỏ hàng: Y rồi X)
    participant DB as Database
    participant InvX as INVENTORY (product X)
    participant InvY as INVENTORY (product Y)

    CustA->>DB: BEGIN Order A checkout
    CustB->>DB: BEGIN Order B checkout

    DB->>InvX: Order A: SELECT ... FOR UPDATE inventory X (theo added_sequence, X trước)
    InvX-->>DB: Lock granted cho Order A

    DB->>InvY: Order B: SELECT ... FOR UPDATE inventory Y (theo added_sequence, Y trước)
    InvY-->>DB: Lock granted cho Order B

    DB->>InvY: Order A: SELECT ... FOR UPDATE inventory Y - chờ lock
    Note over InvY: Inventory Y đang bị Order B giữ lock

    DB->>InvX: Order B: SELECT ... FOR UPDATE inventory X - chờ lock
    Note over InvX: Inventory X đang bị Order A giữ lock

    Note over DB,InvY: Order A chờ Order B, Order B chờ Order A, DEADLOCK
    DB-->>CustA: Lỗi deadlock, transaction rollback, không retry
    DB-->>CustB: Transaction còn lại tiếp tục commit thành công
```
