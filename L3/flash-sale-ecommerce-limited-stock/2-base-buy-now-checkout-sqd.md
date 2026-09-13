# Sequence - Base: Mua ngay (đọc tồn kho rồi trừ, race condition)

Đây là flow **base**: hàng chục nghìn request cùng "Mua ngay" đổ thẳng xuống DB, mỗi request đọc `stock_qty`, kiểm tra `> 0` ở tầng application rồi mới `UPDATE` trừ đi 1. Khi tồn kho còn đúng 1 sản phẩm, 2 request đến gần như đồng thời đều đọc thấy `stock_qty=1 > 0`, cả 2 đều được xác nhận mua, dẫn tới overselling. Đây là tiền đề cho yêu cầu 1, 2 và 3 của đề bài.

```mermaid
sequenceDiagram
    participant U1 as Khách A
    participant U2 as Khách B
    participant API as Checkout API
    participant DB as Database

    Note over DB: stock_qty = 1 (sản phẩm cuối cùng)
    par 2 request gần như đồng thời
        U1->>API: Bấm Mua ngay
        API->>DB: SELECT stock_qty FROM PRODUCT (=1)
    and
        U2->>API: Bấm Mua ngay
        API->>DB: SELECT stock_qty FROM PRODUCT (=1, chưa thấy thay đổi)
    end
    Note over API: Cả 2 request đều thấy stock_qty=1 > 0, đều nghĩ mình mua được
    API->>DB: UPDATE stock_qty = 0 (cho khách A)
    API->>DB: Tạo ORDER cho khách A, status=confirmed
    API-->>U1: Mua thành công
    API->>DB: UPDATE stock_qty = -1 (cho khách B, ghi đè, không kiểm tra lại)
    API->>DB: Tạo ORDER cho khách B, status=confirmed
    API-->>U2: Mua thành công
    Note over DB: Overselling, 2 khách cùng mua được sản phẩm cuối cùng, stock_qty âm
```
