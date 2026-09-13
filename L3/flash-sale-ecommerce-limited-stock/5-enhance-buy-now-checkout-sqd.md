# Sequence - Enhance: Mua ngay (atomic update, kiểm tra affected rows)

Đây là flow **enhance** của `buy-now-checkout` (so với base). Khác biệt so với base: không còn bước `SELECT` riêng để kiểm tra tồn kho, thay vào đó chạy thẳng `UPDATE stock_qty = stock_qty - 1 WHERE stock_qty > 0` và kiểm tra số dòng bị ảnh hưởng (affected rows) để biết có mua được hay không — DB tự đảm bảo tính atomic ở mức row-lock, nên khi tồn kho còn đúng 1, trong 2 request đến gần như đồng thời chỉ đúng 1 request có `affected_rows=1`. Đáp ứng **yêu cầu 1 và 3** của đề bài.

```mermaid
sequenceDiagram
    participant U1 as Khách A
    participant U2 as Khách B
    participant API as Checkout API
    participant DB as Database

    Note over DB: stock_qty = 1 (sản phẩm cuối cùng)
    par 2 request gần như đồng thời
        U1->>API: Bấm Mua ngay
        API->>DB: UPDATE PRODUCT SET stock_qty = stock_qty - 1 WHERE product_id=X AND stock_qty > 0
    and
        U2->>API: Bấm Mua ngay
        API->>DB: UPDATE PRODUCT SET stock_qty = stock_qty - 1 WHERE product_id=X AND stock_qty > 0
    end
    Note over DB: DB serialize 2 UPDATE trên cùng 1 row, chỉ 1 câu lệnh thấy stock_qty=1 và trừ được, câu còn lại thấy stock_qty=0 nên điều kiện WHERE không khớp
    DB-->>API: affected_rows=1 cho khách A, affected_rows=0 cho khách B
    API->>DB: Tạo ORDER cho khách A, status=confirmed
    API-->>U1: Mua thành công
    API-->>U2: Hết hàng (phản hồi ngay, không phải timeout)
    Note over DB: Đúng 1 khách mua được sản phẩm cuối, không overselling
```
