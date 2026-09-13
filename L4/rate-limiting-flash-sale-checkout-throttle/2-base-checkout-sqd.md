# Base sequence — Checkout (gọi thẳng backend, không có admission control)

Đây là **base**, flow "Checkout" ở trạng thái hiện tại — request đi thẳng từ client vào backend (DB, payment) không qua bất kỳ điểm kiểm soát lượng vào nào. Flow này liên quan mật thiết tới enhance vì toàn bộ vấn đề của đề bài (backend quá tải, không công bằng khi nghẽn, không đo lường được) đều bắt nguồn từ việc thiếu tầng throttle ở đây.

```mermaid
sequenceDiagram
    actor UserA as User A
    actor UserB as User B
    actor UserN as ... hàng nghìn user khác
    participant Checkout as Checkout Service
    participant DB as Order/Inventory DB
    participant Payment as Payment Service

    Note over UserA,UserN: Giờ flash sale, tất cả cùng bấm "Mua ngay" trong vài giây

    par Toàn bộ request dồn thẳng vào backend cùng lúc
        UserA->>Checkout: POST /checkout
        Checkout->>DB: Kiểm tra + trừ flash_sale_stock, tạo ORDER
    and
        UserB->>Checkout: POST /checkout
        Checkout->>DB: Kiểm tra + trừ flash_sale_stock, tạo ORDER
    and
        UserN->>Checkout: POST /checkout
        Checkout->>DB: Kiểm tra + trừ flash_sale_stock, tạo ORDER
    end

    DB-->>Checkout: Latency tăng cao do quá nhiều connection đồng thời
    Note over DB: Không có cơ chế nào giới hạn số request được vào cùng lúc, DB có thể timeout hoặc sập hoàn toàn

    Checkout->>Payment: Gọi thanh toán (cho các request DB xử lý kịp)
    Checkout-->>UserA: Timeout / lỗi 500
    Checkout-->>UserB: Timeout / lỗi 500

    Note over Checkout,UserN: Không ai biết mình đang "xếp hàng" hay đã thất bại hẳn, không có ước tính thời gian chờ
```
