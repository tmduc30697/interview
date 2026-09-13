# Sequence - Base - Flow "update-quantity"

Đây là **base**: cách seller sửa số lượng tồn kho trước khi có quy tắc phối hợp với reservation đang tồn tại - đơn giản là ghi đè `quantity` mới, không xét tới các reservation đã giữ trước đó. Flow này được chọn vì nó là tiền đề cho yêu cầu 1 (seller sửa số lượng khi buyer đang giữ reservation) và yêu cầu 5 (chưa có audit log nào ghi lại thay đổi).

```mermaid
sequenceDiagram
    actor Seller
    participant App as Marketplace App
    participant DB as Database

    Note over DB: quantity hiện tại = 5, đã có 3 reservation active (tổng 3 đơn vị)

    Seller->>App: sửa số lượng tồn kho từ 5 xuống 2
    App->>DB: ghi đè quantity = 2

    Note over DB: Không có ghi chú ai sửa, khi nào, giá trị trước/sau\nKhông xét reservation đang active, dữ liệu có thể hiển thị "âm"
```
