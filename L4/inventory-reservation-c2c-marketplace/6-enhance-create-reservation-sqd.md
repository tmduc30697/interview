# Sequence - Enhance - Flow "create-reservation"

Đây là **enhance**, cùng flow `create-reservation` như ở base nhưng thay update read-modify-write bằng update nguyên tử có điều kiện trên `reserved_count`, đáp ứng yêu cầu 3: 2 buyer bấm "Mua ngay" gần như đồng thời cho sản phẩm chỉ còn 1 số lượng khả dụng thì chỉ 1 người giữ được reservation. Flow cũng gán `expires_at` theo TTL ngắn của C2C ngay khi tạo, chuẩn bị cho yêu cầu 4.

```mermaid
sequenceDiagram
    actor Buyer1 as Buyer A
    actor Buyer2 as Buyer B
    participant App as Marketplace App
    participant DB as Database

    Note over DB: quantity = 5, reserved_count = 4, còn khả dụng = 1

    par Buyer A bấm Mua ngay
        Buyer1->>App: Mua ngay sản phẩm X
        App->>DB: UPDATE product SET reserved_count = reserved_count + 1\nWHERE id = X AND quantity - reserved_count >= 1
        DB-->>App: 1 dòng bị ảnh hưởng, thành công
        App->>DB: tạo reservation cho Buyer A, expires_at = now + TTL ngắn
        App-->>Buyer1: giữ chỗ thành công
    and Buyer B bấm Mua ngay gần như cùng lúc
        Buyer2->>App: Mua ngay sản phẩm X
        App->>DB: UPDATE product SET reserved_count = reserved_count + 1\nWHERE id = X AND quantity - reserved_count >= 1
        DB-->>App: 0 dòng bị ảnh hưởng, điều kiện không còn đúng
        App-->>Buyer2: hết hàng, không tạo reservation
    end

    Note over DB: Đảm bảo không có 2 reservation cùng tồn tại cho 1 đơn vị hàng,\nxác nhận bằng test giả lập concurrency chạy nhiều lần
```
