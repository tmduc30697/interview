# Sequence - Enhance - Flow "change-variant-selection"

Đây là **enhance**, cùng flow `change-variant-selection` như ở base nhưng gộp nhả reservation cũ và giữ reservation mới vào 1 transaction atomic, đáp ứng yêu cầu 2: không có khoảng hở nào giữa 2 thao tác, và nếu giữ biến thể mới thất bại thì biến thể cũ không bị nhả oan.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Sàn thời trang
    participant VariantM as Variant (size M)
    participant VariantL as Variant (size L)

    Note over VariantM: Customer đang giữ reservation active cho size M

    Customer->>App: đổi lựa chọn từ size M sang size L
    App->>App: BEGIN transaction

    App->>VariantL: UPDATE quantity = quantity - 1 WHERE size = L AND quantity > 0

    alt Giữ size L thành công
        VariantL-->>App: 1 dòng bị ảnh hưởng
        App->>VariantM: nhả reservation cũ, quantity += 1
        App->>App: tạo reservation mới cho size L,\nprevious_reservation_id trỏ về reservation size M cũ
        App->>App: COMMIT transaction
        App-->>Customer: đổi biến thể thành công
    else Size L đã hết ngay tại thời điểm giữ
        VariantL-->>App: 0 dòng bị ảnh hưởng
        App->>App: ROLLBACK transaction, không đụng gì tới reservation size M
        App-->>Customer: báo lỗi size L đã hết, vẫn đang giữ nguyên size M như trước
    end
```
