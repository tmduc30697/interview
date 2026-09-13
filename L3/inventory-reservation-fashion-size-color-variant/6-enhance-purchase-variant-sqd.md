# Sequence - Enhance - Flow "purchase-variant"

Đây là **enhance**, cùng flow `purchase-variant` như ở base nhưng thay việc dùng cache và đọc-rồi-ghi bằng: (1) luôn truy vấn lại tồn kho khả dụng thực của đúng biến thể tại thời điểm bấm mua, không dựa vào trạng thái đã cache lúc load trang (yêu cầu 3), và (2) update nguyên tử khóa theo tổ hợp `(product_id, size, color)` chứ không phải theo product_id chung (yêu cầu 1). Khách thua chỉ nhận báo hết đúng biến thể đó, không phải báo hết cả sản phẩm.

```mermaid
sequenceDiagram
    actor CustomerA as Khách A
    actor CustomerB as Khách B
    participant App as Sàn thời trang
    participant Variant as Variant (product X, size M, màu đen)

    Note over App: Trang sản phẩm hiển thị "còn hàng" tổng quát (chỉ là gợi ý,\nvì các biến thể khác còn nhiều), không phải cam kết riêng biến thể này
    Note over Variant: quantity = 1 cho đúng biến thể size M, màu đen

    par Khách A bấm mua size M, màu đen
        CustomerA->>App: bấm mua
        App->>Variant: UPDATE quantity = quantity - 1\nWHERE product_id = X AND size = M AND color = đen AND quantity > 0
        Variant-->>App: 1 dòng bị ảnh hưởng, thành công
        App->>App: tạo reservation cho Khách A
        App-->>CustomerA: giữ được đúng biến thể, mua thành công
    and Khách B bấm mua cùng biến thể gần như cùng lúc
        CustomerB->>App: bấm mua
        App->>Variant: UPDATE quantity = quantity - 1\nWHERE product_id = X AND size = M AND color = đen AND quantity > 0
        Variant-->>App: 0 dòng bị ảnh hưởng, điều kiện không còn đúng
        App-->>CustomerB: báo hết hàng đúng biến thể size M, màu đen\n(không phải báo hết cả sản phẩm X)
    end
```
