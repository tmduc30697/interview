# Sequence - Base - Flow "purchase-variant"

Đây là **base**: khách bấm mua một biến thể cụ thể, hệ thống dùng lại trạng thái "còn hàng" đã cache từ lúc load trang sản phẩm và cập nhật tồn kho theo kiểu đọc trước - ghi sau, không khóa đúng theo tổ hợp biến thể. Flow này được chọn vì nó là tiền đề cho yêu cầu 1 (race giữa 2 khách chọn cùng 1 biến thể) và yêu cầu 3 (tin vào dữ liệu cache thay vì truy vấn lại tồn kho thực).

```mermaid
sequenceDiagram
    actor CustomerA as Khách A
    actor CustomerB as Khách B
    participant App as Sàn thời trang
    participant Variant as Variant (size M, màu đen)

    Note over App: Trang sản phẩm load lúc trước, cache "còn hàng" cho toàn sản phẩm\n(vì các biến thể khác còn nhiều)
    Note over Variant: quantity = 1 cho đúng biến thể size M, màu đen

    CustomerA->>App: chọn size M, màu đen, bấm mua
    App-->>App: dựa vào trạng thái "còn hàng" đã cache, không truy vấn lại
    App->>Variant: đọc quantity hiện tại (đọc = 1)

    CustomerB->>App: chọn size M, màu đen, bấm mua gần như cùng lúc
    App-->>App: cũng dựa vào cache "còn hàng", không truy vấn lại
    App->>Variant: đọc quantity hiện tại (đọc = 1, chưa thấy thay đổi của A)

    App-->>App: (Khách A) quantity > 0, hợp lệ
    App->>Variant: ghi quantity = 0, tạo reservation cho Khách A

    App-->>App: (Khách B) dựa trên giá trị đã đọc trước đó, cũng hợp lệ
    App->>Variant: ghi quantity = 0, tạo reservation cho Khách B

    Note over Variant: Kết quả sai: 2 reservation cùng tồn tại cho 1 đơn vị của đúng biến thể này
```
