# Sequence - Base - Flow "create-reservation"

Đây là **base**: cách buyer đặt mua (giữ reservation) trước khi có xử lý concurrency, dùng read-modify-write đơn giản trên `quantity`. Flow này được chọn vì nó chính là nơi race condition xảy ra khi enhance yêu cầu 2 buyer bấm "Mua ngay" gần như đồng thời cho sản phẩm chỉ còn 1 số lượng (yêu cầu 3).

```mermaid
sequenceDiagram
    actor Buyer1 as Buyer A
    actor Buyer2 as Buyer B
    participant App as Marketplace App
    participant DB as Database

    Note over Buyer1,Buyer2: Sản phẩm còn quantity = 1, 2 buyer bấm Mua ngay gần như đồng thời

    Buyer1->>App: Mua ngay sản phẩm X
    App->>DB: đọc quantity hiện tại (đọc = 1)

    Buyer2->>App: Mua ngay sản phẩm X
    App->>DB: đọc quantity hiện tại (đọc = 1, chưa thấy thay đổi của Buyer A)

    App-->>App: (Buyer A) kiểm tra quantity > 0, hợp lệ
    App->>DB: ghi quantity = 0, tạo reservation cho Buyer A

    App-->>App: (Buyer B) kiểm tra quantity > 0 dựa trên giá trị đã đọc trước đó, cũng hợp lệ
    App->>DB: ghi quantity = 0, tạo reservation cho Buyer B

    Note over DB: Kết quả sai: 2 reservation cùng tồn tại cho 1 đơn vị hàng
```
