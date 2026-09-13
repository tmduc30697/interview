# Base sequence — Checkout (tin cache, không invalidate khi trừ kho)

Đây là **base**, flow "Đặt hàng" ở trạng thái hiện tại — backend tin thẳng giá khách nhìn thấy lúc submit, và trừ kho xong không chủ động invalidate cache. Flow này liên quan mật thiết tới enhance vì yêu cầu 1 và 4 của đề bài chính là sửa đúng 2 lỗ hổng ở đây.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant DB as PRODUCT / ORDER store
    participant Cache as PRODUCT_CACHE_ENTRY store

    Customer->>App: Submit đơn hàng kèm giá đã thấy trên trang (từ cache)
    App->>DB: Tạo ORDER với charged_price = giá khách gửi lên, không đối chiếu lại
    App->>DB: Trừ stock trực tiếp trong PRODUCT
    DB-->>App: Trừ kho thành công
    App-->>Customer: Đặt hàng thành công
    Note over Cache: Không có bước invalidate PRODUCT_CACHE_ENTRY nào được gọi sau khi trừ kho — cache vẫn hiển thị "còn hàng" và giá cũ cho tới khi TTL tự hết hạn
    Note over App,DB: Nếu giá đã đổi (sale kết thúc) giữa lúc khách xem trang và lúc submit, đơn vẫn được charge theo giá cũ khách gửi lên mà không kiểm tra lại
```
