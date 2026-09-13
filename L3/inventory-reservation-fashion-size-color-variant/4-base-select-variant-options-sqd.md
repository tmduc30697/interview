# Sequence - Base - Flow "select-variant-options"

Đây là **base**: giao diện chọn biến thể tuần tự (chọn màu trước, rồi mới hiện các size còn hàng của màu đó) dựa trên snapshot tồn kho lấy tại thời điểm chọn màu, không kiểm tra lại khi khách chọn xong size. Flow này được chọn vì nó là tiền đề cho yêu cầu 4 - tồn kho biến thể có thể đổi ngay trong lúc khách đang thao tác tuần tự, dẫn tới việc khách đi hết qua bước thanh toán rồi mới phát hiện hết hàng.

```mermaid
sequenceDiagram
    actor Customer
    participant App as Sàn thời trang
    participant Variant as Variant (màu đen, size M)

    Customer->>App: chọn màu đen
    App->>Variant: lấy danh sách size còn hàng của màu đen
    Variant-->>App: size M còn hàng (quantity = 1)
    App-->>Customer: hiển thị size M là lựa chọn hợp lệ

    Note over Variant: Ngay lúc này, khách khác vừa mua hết size M màu đen (quantity = 0)

    Customer->>App: chọn size M (dựa trên danh sách đã hiển thị trước đó)
    App-->>Customer: cho phép đi tiếp, không kiểm tra lại tồn kho tại bước chọn size

    Customer->>App: vào bước thanh toán, bấm xác nhận mua
    App->>Variant: kiểm tra tồn kho lúc này mới phát hiện hết hàng
    App-->>Customer: báo lỗi hết hàng muộn, sau khi khách đã đi hết các bước chọn lựa
```
