# Enhance sequence — View product page (tách cache, bypass khi sắp hết hàng)

Đây là **enhance**, cùng flow "View product page" đã có ở base nhưng nay thay đổi theo yêu cầu 1 và 2 của đề bài: tồn kho đọc qua `INVENTORY_CACHE_ENTRY` với TTL rất ngắn (giây), và sản phẩm dưới `low_stock_threshold` bỏ qua cache hoàn toàn để đảm bảo chính xác.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant PriceCache as PRICE_CACHE_ENTRY store
    participant InvCache as INVENTORY_CACHE_ENTRY store
    participant DB as PRODUCT store

    Customer->>App: Mở trang sản phẩm đang flash sale
    App->>PriceCache: Đọc PRICE_CACHE_ENTRY(product_id)
    PriceCache-->>App: Trả giá hiệu lực (đã được pre-warm sẵn quanh giờ đổi giá, xem flow "Sale boundary prewarm")

    App->>DB: Kiểm tra stock hiện tại có dưới low_stock_threshold không
    alt Stock còn nhiều (trên ngưỡng an toàn)
        App->>InvCache: Đọc INVENTORY_CACHE_ENTRY(product_id) (TTL vài giây)
        InvCache-->>App: Trả stock từ cache (chấp nhận độ trễ vài giây, đổi lại throughput cao)
    else Stock dưới ngưỡng an toàn (sắp hết hàng)
        App->>DB: Bỏ qua cache, đọc thẳng stock từ DB
        DB-->>App: Trả stock chính xác tại thời điểm đọc
        Note over App,DB: Chấp nhận tăng tải DB cho nhóm sản phẩm nhỏ này để đổi lấy độ chính xác, tránh hiển thị "còn hàng" sai khi sắp hết
    end
    App-->>Customer: Hiển thị giá + tồn kho
```
