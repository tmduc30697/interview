# Sequence - Base - Flow "close-product"

Đây là **base**: cách seller ngừng bán một sản phẩm trước khi có quy định ranh giới thời điểm rõ ràng với buyer đang xác nhận đặt mua. Flow này được chọn vì nó là tiền đề cho yêu cầu 2 - đề bài yêu cầu định nghĩa rõ "trước/sau" thời điểm đóng sản phẩm, còn ở base thì hành vi mơ hồ, tùy thời điểm request buyer chạm tới app trước hay sau khi cờ status đổi.

```mermaid
sequenceDiagram
    actor Buyer
    actor Seller
    participant App as Marketplace App
    participant DB as Database

    Note over Buyer: Buyer đã tạo reservation trước đó, đang ở bước xác nhận đặt mua

    Seller->>App: Ngừng bán sản phẩm X
    App->>DB: ghi status = closed

    Buyer->>App: xác nhận đặt mua (dùng reservation đã giữ)
    App->>DB: kiểm tra status sản phẩm

    alt Request buyer tới trước khi status đổi
        App-->>Buyer: xác nhận thành công
    else Request buyer tới sau khi status đổi
        App-->>Buyer: từ chối vì sản phẩm đã đóng, dù reservation vẫn còn hiệu lực
    end

    Note over App,DB: Không có quy tắc rõ ràng nào phân biệt 2 trường hợp trên,\nkết quả phụ thuộc thời điểm request chạm server
```
