# Sequence - Enhance - Flow "select-variant-options"

Đây là **enhance**, cùng flow `select-variant-options` như ở base nhưng thêm bước kiểm tra lại tồn kho thực ngay tại thời điểm khách chọn xong lựa chọn phụ thuộc (size sau khi đã chọn màu), đáp ứng yêu cầu 4: nếu biến thể vừa hết ngay trong lúc khách thao tác tuần tự, lỗi được trả ngay tại bước đó thay vì để khách đi hết qua thanh toán rồi mới báo.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Sàn thời trang
    participant Variant as Variant (màu đen, size M)

    Customer->>App: chọn màu đen
    App->>Variant: lấy danh sách size còn hàng của màu đen (truy vấn live)
    Variant-->>App: size M còn hàng (quantity = 1)
    App-->>Customer: hiển thị size M là lựa chọn hợp lệ

    Note over Variant: Ngay lúc này, khách khác vừa mua hết size M màu đen (quantity = 0)

    Customer->>App: chọn size M
    App->>Variant: truy vấn lại tồn kho khả dụng thực ngay tại bước này,\nkhông dùng lại danh sách đã hiển thị lúc chọn màu
    Variant-->>App: quantity = 0, đã hết ngay lúc này

    App-->>Customer: báo lỗi ngay tại bước chọn size,\nyêu cầu chọn lại size khác hoặc màu khác

    Note over App: Khách không phải đi hết qua bước thanh toán mới biết hết hàng
```
