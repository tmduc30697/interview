# Enhance sequence — View product page (atomic snapshot + chống stampede)

Đây là **enhance**, cùng flow "View product page" đã có ở base nhưng nay thay đổi theo yêu cầu 3 và 4 của đề bài: cache được đọc/ghi như 1 snapshot atomic theo `version` (không bao giờ thấy giá mới nhưng tồn kho cũ), và khi nhiều request cùng miss cache thì chỉ 1 request được rebuild (singleflight), số còn lại chờ hoặc dùng tạm dữ liệu cũ thay vì dội hết xuống DB.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant Cache as PRODUCT_CACHE_ENTRY store
    participant Lock as SINGLEFLIGHT_LOCK store
    participant DB as PRODUCT store
    participant Metric as CACHE_METRIC store

    Customer->>App: Mở trang chi tiết sản phẩm
    App->>Cache: Đọc PRODUCT_CACHE_ENTRY(product_id)
    alt Cache hit
        Cache-->>App: Trả snapshot (đã atomic theo source_version)
        App->>Metric: hit_count += 1
    else Cache miss
        Cache-->>App: Miss
        App->>Metric: miss_count += 1
        App->>Lock: Thử giữ SINGLEFLIGHT_LOCK(cache_key=product_id)
        alt Giữ được lock (request đầu tiên)
            Lock-->>App: Acquired
            App->>DB: Đọc PRODUCT 1 lần duy nhất (price + stock + description + version cùng lúc, atomic)
            DB-->>App: Snapshot nhất quán
            App->>Cache: Ghi PRODUCT_CACHE_ENTRY mới (source_version=version, ttl_seconds ngắn)
            App->>Lock: Release lock
            App-->>Customer: Trả dữ liệu vừa build
        else Không giữ được lock (request khác đang rebuild)
            Lock-->>App: Đang bị giữ
            App->>Cache: Đọc tạm giá trị cache cũ (stale-while-revalidate) nếu còn, hoặc chờ ngắn rồi đọc lại
            Cache-->>App: Trả dữ liệu (cũ hoặc vừa được request đầu cập nhật xong)
            App-->>Customer: Trả dữ liệu, không tự đi đọc DB thêm lần nào
        end
    end
```
