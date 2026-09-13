# Base sequence — View product page

Đây là **base**, flow "Xem trang chi tiết sản phẩm" — nền tảng cho enhance: cache chỉ dựa vào TTL cố định dài, nên khi dữ liệu gốc đổi giữa chừng, request đọc vẫn thấy dữ liệu cũ cho tới khi TTL tự hết hạn.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Cache as PRODUCT_CACHE_ENTRY store
    participant DB as PRODUCT store

    Customer->>App: Mở trang chi tiết sản phẩm
    App->>Cache: Đọc PRODUCT_CACHE_ENTRY(product_id)
    alt Cache hit (chưa hết TTL)
        Cache-->>App: Trả dữ liệu cache
    else Cache miss (chưa có hoặc đã hết TTL)
        Cache-->>App: Miss
        App->>DB: Đọc PRODUCT (price, stock, description)
        DB-->>App: Trả dữ liệu
        App->>Cache: Ghi PRODUCT_CACHE_ENTRY mới, ttl_seconds=3600
    end
    App-->>Customer: Hiển thị trang sản phẩm
    Note over Cache,DB: Nhiều request cùng lúc miss cache (vd sản phẩm best-seller) đều tự đi đọc DB song song, không có cơ chế phối hợp
```
