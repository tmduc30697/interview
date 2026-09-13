# E-commerce tìm kiếm và gợi ý sản phẩm

**Hệ thống:** Một sàn e-commerce với hàng triệu sản phẩm, cần thanh tìm kiếm với autocomplete và kết quả tìm kiếm chính xác theo tên/thuộc tính sản phẩm.

**Vai trò của flow:** Đồng bộ dữ liệu sản phẩm (tên, giá, tồn kho, danh mục) từ database chính vào search index, và phục vụ autocomplete/tìm kiếm với độ trễ thấp.

**Yêu cầu cụ thể:**
- Khi sản phẩm thay đổi giá hoặc hết hàng, search index phải được cập nhật trong vòng vài giây, tránh hiển thị giá cũ hoặc cho phép tìm thấy sản phẩm đã hết hàng mà không có cảnh báo rõ ràng.
- Autocomplete phải trả kết quả trong thời gian rất ngắn (dưới 100ms) và ưu tiên gợi ý sản phẩm đang bán tốt/còn hàng lên trước, không chỉ khớp chuỗi ký tự đơn thuần.
- Index phải xử lý được lỗi chính tả và từ đồng nghĩa phổ biến (ví dụ "tivi"/"tv", dấu tiếng Việt có/không dấu) để không bỏ lỡ kết quả đúng khi người dùng gõ sai/thiếu dấu.
- Khi có cập nhật hàng loạt (bulk update giá dịp sale lớn ảnh hưởng hàng trăm nghìn sản phẩm), pipeline đồng bộ vào index không được làm chậm hoặc gây downtime cho tìm kiếm đang phục vụ người dùng thật.
- Có cơ chế phát hiện và tự phục hồi khi index bị lệch so với database nguồn (ví dụ do lỗi đồng bộ), thông qua job đối soát định kỳ chạy full reindex một phần hoặc toàn phần mà không ảnh hưởng tìm kiếm đang chạy (dùng index song song rồi swap).

---

# ERD — Base (trước khi có search index)

Đây là **base**: mô hình dữ liệu suy luận cho sàn e-commerce *trước khi* có pipeline đồng bộ vào search index riêng. Đề bài giả định đã có `PRODUCT` với tên/giá/tồn kho/danh mục lưu trong database chính — nếu không có sẵn dữ liệu sản phẩm này thì yêu cầu "đồng bộ vào search index" sẽ không có nghĩa. Tìm kiếm ở base được suy luận là chạy trực tiếp trên database chính (query LIKE/filter cơ bản), chưa có autocomplete hay xử lý lỗi chính tả/đồng nghĩa.

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : classifies
    PRODUCT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        int stock_quantity
        datetime updated_at
    }

    CATEGORY {
        string category_id PK
        string name
    }
```

---

<a id="2-base-search-product-sqd"></a>

# Sequence Diagram — Base: Search Product

Đây là **base**, flow "tìm kiếm sản phẩm" chạy trực tiếp trên database chính — tiền đề cho thấy hạn chế mà enhance phải giải quyết: query trực tiếp trên hàng triệu sản phẩm chậm, không có autocomplete, không xử lý lỗi chính tả/đồng nghĩa, và không tự phản ánh ngay khi giá/tồn kho đổi.

```mermaid
sequenceDiagram
    actor Customer
    participant WebApp as Storefront
    participant DB as Product Database

    Customer->>WebApp: Nhập từ khóa tìm kiếm
    WebApp->>DB: SELECT * FROM products WHERE name LIKE %keyword%
    DB-->>WebApp: Danh sách sản phẩm khớp chuỗi (có thể chậm với hàng triệu sản phẩm)
    WebApp-->>Customer: Hiển thị kết quả

    Note over WebApp,DB: Không có autocomplete riêng, không xử lý sai chính tả/đồng nghĩa, độ trễ tăng theo kích thước bảng
```

---

# ERD — Enhance (sau khi có search index)

Đây là **enhance**: ERD base cộng với `PRODUCT_INDEX_DOCUMENT` (tài liệu index phục vụ autocomplete/tìm kiếm, có các trường chuẩn hóa cho lỗi chính tả/đồng nghĩa/dấu tiếng Việt), `SYNC_EVENT` ghi nhận thay đổi cần đồng bộ, `INDEX_ALIAS` để swap giữa 2 index song song khi reindex, và `RECONCILIATION_RUN` đối soát định kỳ. So với base, `PRODUCT` không đổi cấu trúc nhưng mọi thay đổi giờ phát sinh `SYNC_EVENT` để đẩy vào index gần thời gian thực thay vì chỉ tồn tại trong DB chính.

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : classifies
    PRODUCT ||--o{ SYNC_EVENT : "triggers on change"
    SYNC_EVENT ||--o| PRODUCT_INDEX_DOCUMENT : "updates"
    PRODUCT ||--o| PRODUCT_INDEX_DOCUMENT : "mirrored into"
    INDEX_ALIAS ||--o{ PRODUCT_INDEX_DOCUMENT : "points to active index"
    RECONCILIATION_RUN ||--o{ PRODUCT : "samples for comparison"

    PRODUCT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        int stock_quantity
        datetime updated_at
    }

    CATEGORY {
        string category_id PK
        string name
    }

    PRODUCT_INDEX_DOCUMENT {
        string product_id PK
        string name_normalized
        string name_synonyms
        decimal price
        string stock_status
        decimal popularity_score
        string category_id
        datetime indexed_at
    }

    SYNC_EVENT {
        string event_id PK
        string product_id FK
        string change_type
        string status
        datetime created_at
        datetime processed_at
    }

    INDEX_ALIAS {
        string alias_id PK
        string active_index_name
        string standby_index_name
        datetime switched_at
    }

    RECONCILIATION_RUN {
        string run_id PK
        date run_date
        int sample_size
        int mismatch_count
        string status
    }
```

---

# Sequence Diagram — Enhance: Search Product

Đây là **enhance**, flow "tìm kiếm sản phẩm" đã tồn tại ở base ([2-base-search-product-sqd.md](#2-base-search-product-sqd)) nay thay đổi hoàn toàn: query chuyển sang chạy trên `PRODUCT_INDEX_DOCUMENT` thay vì DB chính, autocomplete trả kết quả dưới 100ms, ưu tiên sản phẩm còn hàng/bán tốt, và xử lý được lỗi chính tả/đồng nghĩa/dấu tiếng Việt.

```mermaid
sequenceDiagram
    actor Customer
    participant WebApp as Storefront
    participant Search as Search Service
    participant Index as Product Search Index

    Customer->>WebApp: Gõ "tivi" (không dấu, có thể là "tv" hoặc "ti vi")
    WebApp->>Search: Autocomplete request (keyword, partial)

    Search->>Search: Chuẩn hóa từ khóa, tra bảng đồng nghĩa (tivi = tv = ti vi)
    Search->>Index: Query trên name_normalized + name_synonyms, filter stock_status
    Index-->>Search: Danh sách khớp, kèm popularity_score

    Search->>Search: Xếp hạng ưu tiên sản phẩm còn hàng và bán tốt lên trước
    Search-->>WebApp: Kết quả autocomplete (dưới 100ms)
    WebApp-->>Customer: Hiển thị gợi ý

    Note over Search,Index: Toàn bộ truy vấn chạy trên index, không đụng tới database chính, nên nhanh và ổn định dù có hàng triệu sản phẩm
```

---

# Sequence Diagram — Enhance: Sync Product to Index

Đây là **enhance**, flow hoàn toàn mới: khi sản phẩm đổi giá hoặc hết hàng trong database chính, thay đổi phải được phản ánh vào search index trong vòng vài giây, tránh hiển thị giá cũ hoặc để khách tìm thấy sản phẩm đã hết hàng mà không có cảnh báo.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên vận hành
    participant Admin as Admin/Inventory Service
    participant DB as Product Database
    participant Queue as Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Product Search Index

    Staff->>Admin: Cập nhật giá hoặc đánh dấu hết hàng
    Admin->>DB: UPDATE products SET price/stock_quantity
    DB-->>Admin: Updated
    Admin->>Queue: Publish SYNC_EVENT (product_id, change_type)

    Queue->>Indexer: Deliver event
    Indexer->>DB: Đọc lại dữ liệu mới nhất của product
    DB-->>Indexer: name, price, stock_quantity mới
    Indexer->>Index: Update PRODUCT_INDEX_DOCUMENT (price, stock_status)
    Index-->>Indexer: Indexed
    Indexer->>Queue: Mark event processed

    Note over Queue,Indexer: Toàn bộ chuỗi hoàn tất trong vòng vài giây kể từ khi DB được cập nhật
```

---

# Sequence Diagram — Enhance: Bulk Update Sync

Đây là **enhance**, flow hoàn toàn mới: khi có cập nhật giá hàng loạt dịp sale lớn ảnh hưởng hàng trăm nghìn sản phẩm, pipeline đồng bộ index phải xử lý mà không làm chậm hoặc downtime cho tìm kiếm đang phục vụ người dùng thật.

```mermaid
sequenceDiagram
    actor Staff as Nhân viên vận hành
    participant Admin as Admin/Inventory Service
    participant DB as Product Database
    participant Queue as Sync Event Queue
    participant Indexer as Index Sync Worker (bulk lane)
    participant Index as Product Search Index

    Staff->>Admin: Trigger bulk price update (sale lớn, hàng trăm nghìn sản phẩm)
    Admin->>DB: Batch UPDATE products SET price theo từng lô nhỏ
    Admin->>Queue: Publish SYNC_EVENT hàng loạt, gắn priority = bulk

    Queue->>Indexer: Route các event bulk vào lane riêng, tách khỏi lane sync thời gian thực
    loop for each batch of events
        Indexer->>Index: Bulk update PRODUCT_INDEX_DOCUMENT theo lô, throttled
        Index-->>Indexer: Batch indexed
    end

    Note over Queue,Indexer: Lane bulk chạy song song nhưng có giới hạn tốc độ riêng, không cạnh tranh tài nguyên với lane sync thời gian thực của các sản phẩm khác
    Indexer-->>Admin: Bulk sync completed, tìm kiếm vẫn phục vụ liên tục trong suốt quá trình
```

---

# Sequence Diagram — Enhance: Reindex Reconciliation

Đây là **enhance**, flow hoàn toàn mới: job đối soát định kỳ phát hiện index bị lệch so với database nguồn (do lỗi đồng bộ), tự phục hồi bằng full/partial reindex sang index song song rồi swap alias, không ảnh hưởng tìm kiếm đang chạy.

```mermaid
sequenceDiagram
    participant Job as Reconciliation Job
    participant DB as Product Database
    participant Index as Active Product Index
    participant Standby as Standby Index
    participant Alias as Index Alias Service

    Job->>DB: Lấy mẫu/toàn bộ dữ liệu sản phẩm nguồn
    Job->>Index: Lấy dữ liệu tương ứng đang có trong index
    Job->>Job: So sánh, phát hiện các bản ghi lệch

    alt Có lệch phát hiện
        Job->>Standby: Build lại dữ liệu (full hoặc partial reindex) trên index song song
        Standby-->>Job: Reindex hoàn tất, đã khớp với DB nguồn
        Job->>Alias: Swap alias trỏ từ Active Index sang Standby Index
        Alias-->>Job: Alias switched, tìm kiếm chuyển sang index mới ngay lập tức
        Note over Index,Standby: Trong suốt quá trình reindex, tìm kiếm vẫn phục vụ trên index cũ, không downtime
    else Không lệch
        Job->>Job: Ghi nhận reconciliation run thành công, không cần reindex
    end
```
