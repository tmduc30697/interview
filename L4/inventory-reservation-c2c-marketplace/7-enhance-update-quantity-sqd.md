# Sequence - Enhance - Flow "update-quantity"

Đây là **enhance**, cùng flow `update-quantity` như ở base nhưng thay đổi hành vi cốt lõi, đáp ứng yêu cầu 1 và yêu cầu 5: khi seller sửa `quantity` từ 5 xuống 2 lúc đang có 3 reservation active (tổng 3 đơn vị), các reservation cũ vẫn hợp lệ - không bị hủy tự động; chỉ số lượng còn lại để bán mới (`quantity - reserved_count`) bị giới hạn và được kẹp về 0 thay vì hiển thị số âm. Mọi thay đổi đều ghi audit log.

```mermaid
sequenceDiagram
    actor Seller
    participant App as Marketplace App
    participant DB as Database
    participant AuditLog as Inventory Audit Log

    Note over DB: quantity = 5, reserved_count = 3 (3 reservation active không đổi)

    Seller->>App: sửa số lượng tồn kho từ 5 xuống 2
    App->>DB: UPDATE product SET quantity = 2 WHERE id = X
    App->>AuditLog: ghi log, changed_by = Seller, field = quantity, old_value = 5, new_value = 2

    Note over App,DB: 3 reservation active vẫn giữ nguyên, không tự hủy

    App-->>App: tính số lượng còn lại để bán = max(0, quantity - reserved_count)\n= max(0, 2 - 3) = 0

    App-->>Seller: hiển thị "còn lại để bán: 0", không hiển thị số âm

    Note over App: Buyer mới truy cập sản phẩm cũng thấy còn lại để bán = 0,\nkhông thể tạo reservation mới cho tới khi reservation cũ giải phóng
```
