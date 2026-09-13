# Cache trang chi tiết sản phẩm cho e-commerce

**Hệ thống:** Trang e-commerce cache HTML/dữ liệu trang chi tiết sản phẩm (giá, tồn kho, mô tả) để giảm tải DB cho lượng truy cập lớn.

**Vai trò của flow:** Invalidate đúng cache khi sản phẩm được cập nhật (đổi giá, hết hàng) để không hiển thị thông tin sai cho khách, đồng thời giữ hiệu quả cache cao khi sản phẩm không đổi.

**Yêu cầu cụ thể:**
- Khi giá hoặc tồn kho sản phẩm thay đổi ở hệ thống quản trị, cache liên quan phải được invalidate trong vòng vài giây, không dựa hoàn toàn vào TTL cố định dài (ví dụ TTL 1 giờ là không đủ nhanh cho thay đổi giá).
- Có chiến lược rõ giữa cache invalidation theo event (khi có thay đổi, chủ động xóa) và TTL ngắn dự phòng (để tránh trường hợp event bị lỡ vẫn tự hết hạn sau một khoảng thời gian giới hạn).
- Xử lý race condition: request đọc cache đúng lúc dữ liệu đang được invalidate/update không được thấy trạng thái nửa cũ nửa mới (ví dụ giá mới nhưng tồn kho cũ).
- Với sản phẩm có traffic rất cao (best-seller), invalidate không được gây "cache stampede" (mọi request cùng lúc miss cache dội xuống DB) — cần cơ chế warm lại cache có kiểm soát (single flight hoặc pre-warm).
- Đo lường: cache hit ratio theo loại trang, và độ trễ giữa lúc dữ liệu gốc đổi tới lúc cache phản ánh đúng (staleness window thực tế).

---

# Base ERD — Cache trang sản phẩm chỉ dựa vào TTL cố định

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có invalidation theo event. Đề bài nói cache trang chi tiết sản phẩm (giá, tồn kho, mô tả) với TTL 1 giờ — nên base cần đủ: sản phẩm và 1 cache entry theo product_id với TTL cố định dài. Chưa có entity nào phục vụ invalidation theo event/khoá chống stampede/đo lường — những thứ đó là phần enhance.

```mermaid
erDiagram
    PRODUCT ||--o| PRODUCT_CACHE_ENTRY : "cached as"

    PRODUCT {
        string id PK
        string name
        decimal price
        int stock
        string description
        datetime updated_at
    }
    PRODUCT_CACHE_ENTRY {
        string cache_key PK "product_id"
        string value "price + stock + description"
        datetime cached_at
        int ttl_seconds "3600"
    }
```

---

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

---

# Base sequence — Update product (chỉ dựa TTL, không invalidate)

Đây là **base**, flow "Admin đổi giá/tồn kho" ở trạng thái hiện tại — cập nhật DB xong là kết thúc, không đụng gì tới cache. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng lỗ hổng "chờ TTL" ở đây.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminApp as Admin Panel
    participant DB as PRODUCT store

    Admin->>AdminApp: Đổi giá hoặc tồn kho sản phẩm
    AdminApp->>DB: Cập nhật PRODUCT (price/stock, updated_at)
    DB-->>AdminApp: Cập nhật thành công
    AdminApp-->>Admin: "Cập nhật thành công"
    Note over DB: Không có bước invalidate cache nào được gọi — PRODUCT_CACHE_ENTRY vẫn giữ giá trị cũ cho tới khi ttl_seconds=3600 tự hết hạn, khách hàng có thể thấy giá sai tới 1 giờ
```

---

# Enhance ERD — sau khi có invalidation theo event + chống stampede

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 4 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `PRODUCT` thêm `version` — dùng để xây snapshot cache atomic (giá + tồn kho cùng 1 version), tránh trạng thái nửa cũ nửa mới.
- `PRODUCT_CACHE_ENTRY` thêm `source_version` và `ttl_seconds` rút ngắn (dự phòng) — kết hợp cả invalidate theo event lẫn TTL ngắn thay vì chỉ dựa TTL dài như base.
- `CACHE_INVALIDATION_EVENT` (mới) — phát sinh ngay khi giá/tồn kho đổi, invalidate trong vài giây.
- `SINGLEFLIGHT_LOCK` (mới) — chỉ 1 request được rebuild cache khi miss, chống cache stampede.
- `CACHE_METRIC` + `STALENESS_METRIC` (mới) — đo hit ratio theo loại trang và độ trễ invalidate thực tế.

```mermaid
erDiagram
    PRODUCT ||--o| PRODUCT_CACHE_ENTRY : "cached as"
    PRODUCT ||--o{ CACHE_INVALIDATION_EVENT : triggers
    PRODUCT ||--o{ STALENESS_METRIC : measures
    PRODUCT_CACHE_ENTRY ||--o| SINGLEFLIGHT_LOCK : "guarded by (khi rebuild)"

    PRODUCT {
        string id PK
        string name
        decimal price
        int stock
        string description
        int version
        datetime updated_at
    }
    PRODUCT_CACHE_ENTRY {
        string cache_key PK "product_id"
        string value "price + stock + description (snapshot atomic theo version)"
        int source_version
        datetime cached_at
        int ttl_seconds "30-60, dự phòng"
    }
    CACHE_INVALIDATION_EVENT {
        string id PK
        string product_id FK
        string changed_fields "price | stock | both"
        int source_version
        datetime triggered_at
    }
    SINGLEFLIGHT_LOCK {
        string id PK
        string cache_key
        string holder_request_id
        datetime acquired_at
        datetime expires_at
    }
    CACHE_METRIC {
        string id PK
        string page_type "product_detail"
        datetime window_start
        datetime window_end
        int hit_count
        int miss_count
    }
    STALENESS_METRIC {
        string id PK
        string product_id FK
        datetime changed_at
        datetime cache_consistent_at
        int staleness_ms
    }
```

---

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

---

# Enhance sequence — Update product (invalidate theo event + TTL ngắn dự phòng + đo staleness)

Đây là **enhance**, cùng flow "Update product" đã có ở base nhưng nay thay đổi theo yêu cầu 1, 2 và 5 của đề bài: cập nhật giá/tồn kho phát ngay `CACHE_INVALIDATION_EVENT` để invalidate trong vài giây, kết hợp TTL ngắn dự phòng phòng khi event bị lỡ, và đo staleness thực tế.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminApp as Admin Panel
    participant DB as PRODUCT store
    participant EventBus as Cache Invalidation Event
    participant Cache as PRODUCT_CACHE_ENTRY store
    participant Metric as STALENESS_METRIC store

    Admin->>AdminApp: Đổi giá hoặc tồn kho sản phẩm
    AdminApp->>DB: Cập nhật PRODUCT (price/stock, version += 1, updated_at)
    DB-->>AdminApp: Cập nhật thành công (version mới)
    AdminApp->>EventBus: Publish CACHE_INVALIDATION_EVENT (product_id, changed_fields, source_version=version mới)
    EventBus->>Cache: Xoá/đánh dấu invalid PRODUCT_CACHE_ENTRY(product_id) ngay lập tức
    Cache-->>EventBus: Đã invalidate (trong vài giây kể từ lúc đổi)
    EventBus->>Metric: Ghi STALENESS_METRIC (changed_at=updated_at, cache_consistent_at=now)
    AdminApp-->>Admin: "Cập nhật thành công"

    Note over Cache: TTL ngắn (30-60s) vẫn được giữ làm dự phòng — nếu vì lý do gì đó event bị lỡ (mất kết nối, consumer down), cache vẫn tự hết hạn sau khoảng ngắn thay vì kẹt cả giờ như base
    Note over Metric: Request đọc kế tiếp sẽ miss cache và rebuild atomic theo version mới (xem flow "View product page")
```
