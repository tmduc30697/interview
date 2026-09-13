# Cache giá/tồn kho trong flash sale với tần suất thay đổi cực cao

**Hệ thống:** E-commerce trong giờ flash sale có giá và tồn kho thay đổi liên tục (giảm giá theo giờ, tồn kho giảm nhanh), cache phải theo kịp thay đổi mà vẫn chịu được tải đọc khổng lồ.

**Vai trò của flow:** Invalidate cache tồn kho/giá gần như real-time trong khi vẫn giữ được throughput đọc cao, khác với cache thông thường có thể chấp nhận độ trễ lớn hơn.

**Yêu cầu cụ thể:**
- Cache tồn kho phải có TTL rất ngắn (giây) kết hợp active invalidation khi có đơn hàng trừ kho, để giảm thiểu rủi ro hiển thị "còn hàng" sai trong lúc sản phẩm sắp hết.
- Với sản phẩm sắp hết hàng (dưới ngưỡng an toàn), phải chuyển sang đọc trực tiếp từ nguồn (bỏ qua cache) để đảm bảo chính xác, đánh đổi tăng tải DB cho nhóm sản phẩm nhỏ này để bù lại độ chính xác.
- Xử lý cache stampede khi giá flash sale bắt đầu/kết thúc đúng giờ (hàng loạt cache cùng invalidate đồng thời tại thời điểm giờ chốt) — cần pre-warm cache trước thời điểm đổi giá vài giây để tránh dội tải.
- Đảm bảo giá hiển thị cho user tại thời điểm checkout khớp với giá thực tế lúc submit đơn (double-check giá ở backend, không tin hoàn toàn giá đã cache hiển thị ở client).
- Đo lường: tỉ lệ hiển thị sai giá/tồn kho (nếu có) trong log thực tế của các lần flash sale trước, dùng làm baseline để cải thiện chiến lược invalidation.

---

# Base ERD — Cache flash sale trước khi tối ưu cho tần suất thay đổi cực cao

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có các cơ chế đặc thù cho flash sale. Đề bài nói flash sale đã có lịch giảm giá theo giờ và giá/tồn kho hiển thị dựa vào cache — nên base cần đủ: sản phẩm (kèm lịch sale), 1 cache entry chung cho giá+tồn kho với TTL cố định, và đơn hàng. Chưa có entity nào phục vụ invalidation chủ động theo đơn hàng/bypass cache khi sắp hết hàng/pre-warm theo lịch/double-check giá — những thứ đó là phần enhance.

```mermaid
erDiagram
    PRODUCT ||--o| PRODUCT_CACHE_ENTRY : "cached as"
    PRODUCT ||--o{ ORDER : "ordered via"

    PRODUCT {
        string id PK
        string name
        decimal price
        decimal sale_price
        int stock
        datetime sale_start_at
        datetime sale_end_at
    }
    PRODUCT_CACHE_ENTRY {
        string cache_key PK "product_id"
        string value "price hiệu lực + stock"
        datetime cached_at
        int ttl_seconds "vd 30-60"
    }
    ORDER {
        string id PK
        string product_id FK
        int quantity
        decimal charged_price
        datetime created_at
    }
```

---

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

---

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

---

# Enhance ERD — sau khi tối ưu cache cho flash sale

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 5 thay đổi chính, ứng trực tiếp với 5 yêu cầu trong đề bài:

- Tách `PRICE_CACHE_ENTRY` và `INVENTORY_CACHE_ENTRY` riêng (thay vì gộp chung như base) — tồn kho cần TTL rất ngắn + invalidate chủ động, còn giá chủ yếu đổi theo lịch nên có thể pre-warm.
- `INVENTORY_INVALIDATION_EVENT` (mới) — phát sinh ngay khi có đơn hàng trừ kho.
- `PRODUCT` thêm `low_stock_threshold` — sản phẩm dưới ngưỡng này bỏ qua cache, đọc thẳng DB.
- `SALE_WARMUP_JOB` (mới) — pre-warm cache trước thời điểm đổi giá theo lịch vài giây, chống stampede.
- `CHECKOUT_PRICE_CHECK` (mới) — double-check giá ở backend trước khi charge.
- `DISPLAY_ACCURACY_LOG` (mới) — log các lần hiển thị sai giá/tồn kho để làm baseline đo lường.

```mermaid
erDiagram
    PRODUCT ||--o| PRICE_CACHE_ENTRY : "cached as"
    PRODUCT ||--o| INVENTORY_CACHE_ENTRY : "cached as"
    PRODUCT ||--o{ INVENTORY_INVALIDATION_EVENT : triggers
    PRODUCT ||--o{ SALE_WARMUP_JOB : "scheduled for"
    PRODUCT ||--o{ ORDER : "ordered via"
    PRODUCT ||--o{ DISPLAY_ACCURACY_LOG : "measured on"
    ORDER ||--o| CHECKOUT_PRICE_CHECK : "verified by"
    ORDER ||--o| INVENTORY_INVALIDATION_EVENT : "causes"

    PRODUCT {
        string id PK
        string name
        decimal price
        decimal sale_price
        int stock
        int low_stock_threshold
        datetime sale_start_at
        datetime sale_end_at
    }
    PRICE_CACHE_ENTRY {
        string cache_key PK "product_id"
        decimal price_value
        datetime cached_at
        int ttl_seconds
    }
    INVENTORY_CACHE_ENTRY {
        string cache_key PK "product_id"
        int stock_value
        datetime cached_at
        int ttl_seconds "rất ngắn, vài giây"
    }
    INVENTORY_INVALIDATION_EVENT {
        string id PK
        string product_id FK
        string order_id FK
        datetime triggered_at
    }
    SALE_WARMUP_JOB {
        string id PK
        string product_id FK
        datetime scheduled_change_at
        datetime warmup_started_at
        string status
    }
    ORDER {
        string id PK
        string product_id FK
        int quantity
        decimal displayed_price
        decimal charged_price
        datetime created_at
    }
    CHECKOUT_PRICE_CHECK {
        string id PK
        string order_id FK
        decimal displayed_price
        decimal authoritative_price
        boolean match
        datetime checked_at
    }
    DISPLAY_ACCURACY_LOG {
        string id PK
        string product_id FK
        string metric_type "price_mismatch | stock_oversell_risk"
        string displayed_value
        string actual_value
        datetime detected_at
    }
```

---

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

---

# Enhance sequence — Checkout (invalidate chủ động + double-check giá)

Đây là **enhance**, cùng flow "Checkout" đã có ở base nhưng nay thay đổi theo yêu cầu 1, 4 và 5 của đề bài: trừ kho xong invalidate ngay `INVENTORY_CACHE_ENTRY` thay vì chờ TTL, và backend đối chiếu lại giá thực tế tại thời điểm submit thay vì tin thẳng giá client gửi lên.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant DB as PRODUCT / ORDER store
    participant InvCache as INVENTORY_CACHE_ENTRY store
    participant Log as DISPLAY_ACCURACY_LOG store

    Customer->>App: Submit đơn hàng kèm giá đã thấy trên trang (displayed_price)
    App->>DB: Đọc authoritative_price hiện tại của sản phẩm (theo lịch sale tại thời điểm submit)
    DB-->>App: Trả authoritative_price
    App->>App: So sánh displayed_price với authoritative_price
    alt Khớp nhau
        App->>DB: Tạo ORDER với charged_price = authoritative_price
    else Không khớp (giá đã đổi giữa lúc xem trang và lúc submit)
        App->>Log: Ghi DISPLAY_ACCURACY_LOG (metric_type=price_mismatch, displayed_value, actual_value)
        App->>DB: Tạo ORDER với charged_price = authoritative_price (không tin giá cũ khách gửi lên)
        App-->>Customer: Thông báo giá đã cập nhật, xác nhận lại trước khi charge nếu chênh lệch lớn
    end
    App->>DB: Trừ stock trực tiếp trong PRODUCT
    DB-->>App: Trừ kho thành công
    App->>InvCache: Publish INVENTORY_INVALIDATION_EVENT → invalidate INVENTORY_CACHE_ENTRY(product_id) ngay lập tức
    InvCache-->>App: Đã invalidate (không chờ TTL vài giây tự hết hạn)
    App-->>Customer: Đặt hàng thành công với giá chính xác
```

---

# Enhance sequence — Sale boundary prewarm

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 3 của đề bài: pre-warm cache trước thời điểm giờ flash sale bắt đầu/kết thúc vài giây, tránh hàng loạt cache cùng miss đồng thời đúng lúc chốt giờ (cache stampede).

```mermaid
sequenceDiagram
    participant Scheduler as Sale Schedule Scheduler
    participant Job as SALE_WARMUP_JOB store
    participant DB as PRODUCT store
    participant PriceCache as PRICE_CACHE_ENTRY store

    Scheduler->>Scheduler: Theo dõi sale_start_at/sale_end_at của các sản phẩm sắp tới
    Scheduler->>Job: Tạo SALE_WARMUP_JOB (scheduled_change_at, warmup_started_at = scheduled_change_at - vài giây)

    Note over Job: Đợi tới warmup_started_at (vài giây trước giờ chốt)

    Job->>DB: Tính trước giá hiệu lực sẽ áp dụng ngay sau thời điểm chốt (sale_price hoặc giá gốc)
    Job->>PriceCache: Ghi sẵn PRICE_CACHE_ENTRY mới cho giá sắp có hiệu lực, cached_at đặt đúng thời điểm chuyển
    PriceCache-->>Job: Cache đã sẵn sàng trước giờ chốt

    Note over Job,PriceCache: Khi đồng hồ chạm sale_start_at/sale_end_at, cache đã có sẵn giá trị mới — request đọc ngay sau đó đều hit cache, không có làn sóng miss đồng thời dội xuống DB

    Job->>Job: status=completed
```
