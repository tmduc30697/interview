# Sequence - Base - Flow "checkout-reserve"

Đây là **base**: khách vào bước checkout, hệ thống chọn kho gần nhất rồi giữ tồn kho bằng cách đọc số dư trước, kiểm tra, rồi ghi sau (2 bước tách rời, không atomic). Flow này được chọn vì nó chính là nguyên nhân của race mà yêu cầu 1 mô tả: 2 khách ở 2 khu vực khác nhau cùng checkout sản phẩm chỉ còn 1 đơn vị ở kho gần cả hai.

```mermaid
sequenceDiagram
    actor CustomerA as Khách A (khu vực gần kho X)
    actor CustomerB as Khách B (khu vực gần kho X)
    participant App as E-commerce App
    participant Inventory as Inventory (kho X)

    Note over Inventory: available = 1, reserved = 0 tại kho X

    CustomerA->>App: vào checkout sản phẩm Y
    App->>App: chọn kho gần nhất, tìm được kho X
    App->>Inventory: đọc available - reserved (đọc = 1)

    CustomerB->>App: vào checkout sản phẩm Y gần như cùng lúc
    App->>App: chọn kho gần nhất, cũng là kho X
    App->>Inventory: đọc available - reserved (đọc = 1, chưa thấy thay đổi của A)

    App-->>App: (Khách A) 1 > 0, hợp lệ
    App->>Inventory: ghi reserved = reserved + 1

    App-->>App: (Khách B) dựa trên giá trị đã đọc trước đó, cũng hợp lệ
    App->>Inventory: ghi reserved = reserved + 1

    Note over Inventory: reserved = 2 nhưng available chỉ có 1,\ncả 2 khách đều tưởng giữ được hàng
```
