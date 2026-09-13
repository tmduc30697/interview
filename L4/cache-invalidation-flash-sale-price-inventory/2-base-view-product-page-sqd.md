# Base sequence — View product page (TTL cố định, chưa tách stock riêng)

Đây là **base**, flow "Xem trang sản phẩm trong giờ flash sale" — dùng chung 1 cache entry cho cả giá lẫn tồn kho với TTL cố định, giống cache thông thường. Đây chính là điểm mà yêu cầu 1 và 2 của đề bài nhắm vào: tồn kho biến động quá nhanh so với TTL này.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Cache as PRODUCT_CACHE_ENTRY store
    participant DB as PRODUCT store

    Customer->>App: Mở trang sản phẩm đang flash sale
    App->>Cache: Đọc PRODUCT_CACHE_ENTRY(product_id)
    alt Cache hit
        Cache-->>App: Trả giá + tồn kho từ cache
    else Cache miss
        Cache-->>App: Miss
        App->>DB: Đọc PRODUCT (price/sale_price theo lịch, stock)
        DB-->>App: Trả dữ liệu
        App->>Cache: Ghi lại cache, ttl_seconds cố định
    end
    App-->>Customer: Hiển thị giá + "còn hàng/hết hàng"
    Note over Cache,DB: Không phân biệt sản phẩm sắp hết hàng, tất cả đều đọc qua cùng 1 cache với TTL như nhau
```
