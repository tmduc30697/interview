# Marketplace đa category với facet lọc khác biệt hoàn toàn

**Hệ thống:** Một marketplace bán hàng đa category hoàn toàn khác biệt (điện tử, thời trang, nội thất...), mỗi category có bộ thuộc tính/facet lọc riêng.

**Vai trò của flow:** Lập chỉ mục sản phẩm từ nhiều category với schema thuộc tính khác nhau, phục vụ tìm kiếm xuyên category và autocomplete gợi ý đúng theo ngữ cảnh category đang tìm.

**Yêu cầu cụ thể:**
- Mỗi category có bộ facet lọc riêng (điện tử có "dung lượng RAM", thời trang có "size/màu", nội thất có "chất liệu") — index phải lưu và truy vấn hiệu quả với schema thuộc tính động theo category mà không phải tạo index riêng biệt cho từng category gây khó khăn khi tìm xuyên category.
- Khi người dùng gõ một từ khóa mơ hồ có thể thuộc nhiều category (ví dụ "bàn" có thể là bàn ăn nội thất hoặc liên quan đến bàn phím máy tính), autocomplete phải suy luận được ngữ cảnh category phù hợp dựa trên hành vi tìm kiếm/lịch sử, tránh gợi ý lẫn lộn không liên quan gây khó chịu.
- Khi sản phẩm được người bán đổi category sau khi đã đăng, index phải cập nhật đồng thời cả danh mục lẫn bộ facet thuộc tính tương ứng, đảm bảo sản phẩm không còn xuất hiện ở bộ lọc facet của category cũ không còn phù hợp.
- Kết quả tìm kiếm xuyên category (ví dụ tìm "quà tặng sinh nhật" trả về cả điện tử, thời trang, đồ chơi) phải xếp hạng công bằng giữa các category khác nhau về bản chất — độ khớp từ khóa của một sản phẩm thời trang không thể so trực tiếp với một sản phẩm điện tử theo cùng công thức điểm số mà không tính đến đặc thù từng category.
- Khi thêm một category hoàn toàn mới vào hệ thống, việc bổ sung schema/facet mới cho category đó không được yêu cầu reindex toàn bộ dữ liệu các category hiện có đang phục vụ tìm kiếm sống.

---

# ERD — Base (trước khi hợp nhất index đa category)

Đây là **base**: mô hình dữ liệu suy luận cho marketplace *trước khi* hợp nhất facet đa category. Đề bài chỉ ra điểm cần tránh là "phải tạo index riêng biệt cho từng category gây khó khăn khi tìm xuyên category" — suy ra base chính là trạng thái đó: mỗi category có 1 index riêng với schema thuộc tính cố định cho category đó, khiến tìm xuyên category khó khăn. Đây là tiền đề để yêu cầu "index thống nhất, schema thuộc tính động" có nghĩa.

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : classifies
    CATEGORY ||--|| CATEGORY_INDEX : "has its own separate index"
    PRODUCT ||--o| CATEGORY_INDEX_DOCUMENT : "indexed only into its category's index"

    CATEGORY {
        string category_id PK
        string name
    }

    PRODUCT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        string attributes_json
    }

    CATEGORY_INDEX {
        string index_id PK
        string category_id FK
        string fixed_attribute_schema
    }

    CATEGORY_INDEX_DOCUMENT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        string category_specific_attributes
    }
```

---

<a id="2-base-index-product-sqd"></a>

# Sequence Diagram — Base: Index Product

Đây là **base**, flow "lập chỉ mục sản phẩm" khi mỗi category có 1 index riêng biệt — tiền đề cho thấy hạn chế: sản phẩm chỉ được index vào đúng 1 index của category nó thuộc về, theo schema cố định của category đó.

```mermaid
sequenceDiagram
    actor Seller
    participant ProductSvc as Product Service
    participant DB as Product Database
    participant Indexer as Category Index Worker

    Seller->>ProductSvc: Đăng sản phẩm mới (category = electronics)
    ProductSvc->>DB: INSERT INTO products (category_id, attributes_json)
    DB-->>ProductSvc: product_id created
    ProductSvc->>Indexer: Sync sang index của category electronics

    Indexer->>Indexer: Áp fixed_attribute_schema của category electronics
    Indexer-->>ProductSvc: Indexed vào electronics_index

    Note over Indexer: Sản phẩm chỉ tồn tại trong index riêng của category electronics, không nằm trong index chung nào để tìm xuyên category
```

---

<a id="3-base-search-product-sqd"></a>

# Sequence Diagram — Base: Search Product

Đây là **base**, flow "tìm kiếm sản phẩm" khi mỗi category có index riêng — cho thấy rõ vấn đề: tìm trong 1 category hoạt động tốt, nhưng tìm xuyên nhiều category (ví dụ "quà tặng sinh nhật") phải query nhiều index riêng lẻ rồi tự gộp kết quả, khó xếp hạng công bằng.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Marketplace App
    participant Search as Search Service
    participant ElectronicsIdx as Electronics Index
    participant FashionIdx as Fashion Index
    participant FurnitureIdx as Furniture Index

    Buyer->>WebApp: Tìm "quà tặng sinh nhật"
    WebApp->>Search: Search request (keyword, không rõ category)

    par Query song song từng index riêng biệt
        Search->>ElectronicsIdx: Query theo schema electronics
        ElectronicsIdx-->>Search: Kết quả + điểm khớp riêng của electronics
    and
        Search->>FashionIdx: Query theo schema fashion
        FashionIdx-->>Search: Kết quả + điểm khớp riêng của fashion
    and
        Search->>FurnitureIdx: Query theo schema furniture
        FurnitureIdx-->>Search: Kết quả + điểm khớp riêng của furniture
    end

    Search->>Search: Tự gộp kết quả từ nhiều index, khó so điểm khớp công bằng giữa các category
    Search-->>WebApp: Kết quả gộp, có thể lệch ưu tiên category
    WebApp-->>Buyer: Hiển thị kết quả
```

---

# ERD — Enhance (sau khi hợp nhất index đa category)

Đây là **enhance**: thay vì mỗi category 1 index riêng (base), nay chỉ còn 1 `PRODUCT_INDEX` thống nhất với schema thuộc tính động qua `PRODUCT_FACET_VALUE` (mô hình EAV), mỗi category định nghĩa facet của mình qua `CATEGORY_FACET_SCHEMA` mà không cần đổi cấu trúc index. Thêm `SEARCH_BEHAVIOR_LOG` phục vụ suy luận ngữ cảnh category cho autocomplete từ khóa mơ hồ.

```mermaid
erDiagram
    CATEGORY ||--o{ PRODUCT : classifies
    CATEGORY ||--o{ CATEGORY_FACET_SCHEMA : defines
    PRODUCT ||--o| PRODUCT_INDEX_DOCUMENT : "indexed into single unified index"
    PRODUCT_INDEX_DOCUMENT ||--o{ PRODUCT_FACET_VALUE : "has dynamic facet values"
    CATEGORY_FACET_SCHEMA ||--o{ PRODUCT_FACET_VALUE : "constrains"
    USER ||--o{ SEARCH_BEHAVIOR_LOG : generates

    CATEGORY {
        string category_id PK
        string name
    }

    PRODUCT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        string attributes_json
    }

    CATEGORY_FACET_SCHEMA {
        string schema_id PK
        string category_id FK
        string facet_key
        string facet_type
    }

    PRODUCT_INDEX_DOCUMENT {
        string product_id PK
        string category_id FK
        string name
        decimal price
        decimal category_normalized_score
    }

    PRODUCT_FACET_VALUE {
        string facet_value_id PK
        string product_id FK
        string facet_key
        string facet_value
    }

    USER {
        string user_id PK
    }

    SEARCH_BEHAVIOR_LOG {
        string log_id PK
        string user_id FK
        string query_term
        string selected_category_id
        datetime searched_at
    }
```

---

# Sequence Diagram — Enhance: Index Product

Đây là **enhance**, flow "lập chỉ mục sản phẩm" đã tồn tại ở base ([2-base-index-product-sqd.md](#2-base-index-product-sqd)) nay thay đổi cốt lõi: sản phẩm được index vào duy nhất 1 `PRODUCT_INDEX` thống nhất, thuộc tính riêng của category được lưu dưới dạng `PRODUCT_FACET_VALUE` động tra theo `CATEGORY_FACET_SCHEMA`, không cần index riêng biệt cho từng category.

```mermaid
sequenceDiagram
    actor Seller
    participant ProductSvc as Product Service
    participant DB as Product Database
    participant Indexer as Unified Index Worker
    participant SchemaStore as Category Facet Schema
    participant Index as Unified Product Index

    Seller->>ProductSvc: Đăng sản phẩm mới (category = electronics, attributes_json)
    ProductSvc->>DB: INSERT INTO products (category_id, attributes_json)
    DB-->>ProductSvc: product_id created
    ProductSvc->>Indexer: Sync sang unified index

    Indexer->>SchemaStore: Lấy facet schema của category electronics (RAM, màn hình...)
    SchemaStore-->>Indexer: Danh sách facet_key hợp lệ cho electronics

    Indexer->>Index: Ghi PRODUCT_INDEX_DOCUMENT (name, price, category_id)
    Indexer->>Index: Ghi PRODUCT_FACET_VALUE cho từng thuộc tính theo schema động
    Index-->>Indexer: Indexed

    Note over Index: Sản phẩm nằm trong cùng 1 index với mọi category khác, tìm xuyên category không cần gộp nhiều nguồn
```

---

# Sequence Diagram — Enhance: Search Product

Đây là **enhance**, flow "tìm kiếm sản phẩm" đã tồn tại ở base ([3-base-search-product-sqd.md](#3-base-search-product-sqd)) nay thay đổi cốt lõi: chỉ 1 query duy nhất trên unified index thay vì query song song nhiều index riêng, và điểm khớp được chuẩn hóa theo đặc thù từng category trước khi so sánh, đảm bảo xếp hạng công bằng xuyên category.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Marketplace App
    participant Search as Search Service
    participant Index as Unified Product Index

    Buyer->>WebApp: Tìm "quà tặng sinh nhật"
    WebApp->>Search: Search request (keyword)

    Search->>Index: Query 1 lần duy nhất trên unified index, xuyên mọi category
    Index-->>Search: Kết quả kèm điểm khớp thô theo từng category

    Search->>Search: Chuẩn hóa điểm khớp theo đặc thù từng category (category_normalized_score) trước khi so sánh
    Search->>Search: Xếp hạng công bằng dựa trên điểm đã chuẩn hóa, không so trực tiếp điểm thô

    Search-->>WebApp: Kết quả xuyên category đã xếp hạng công bằng
    WebApp-->>Buyer: Hiển thị cả điện tử, thời trang, đồ chơi... xen kẽ hợp lý
```

---

# Sequence Diagram — Enhance: Autocomplete Ambiguous Term

Đây là **enhance**, flow hoàn toàn mới: khi từ khóa gõ vào có thể thuộc nhiều category (ví dụ "bàn"), autocomplete suy luận ngữ cảnh category phù hợp dựa trên hành vi tìm kiếm/lịch sử của người dùng, tránh gợi ý lẫn lộn.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Marketplace App
    participant Search as Search Service
    participant BehaviorLog as Search Behavior Log
    participant Index as Unified Product Index

    Buyer->>WebApp: Gõ "bàn" (có thể là bàn ăn hoặc bàn phím)
    WebApp->>Search: Autocomplete request (partial keyword, user_id)

    Search->>BehaviorLog: Tra lịch sử tìm kiếm/category đã chọn gần đây của user này
    BehaviorLog-->>Search: User trước đó hay tìm nội thất, ít khi tìm điện tử

    Search->>Index: Query "bàn" trên unified index, ưu tiên boost theo category nội thất
    Index-->>Search: Gợi ý autocomplete (ưu tiên bàn ăn, bàn trà... lên trước bàn phím)

    Search-->>WebApp: Danh sách gợi ý đã theo ngữ cảnh
    WebApp-->>Buyer: Hiển thị gợi ý phù hợp với thói quen tìm kiếm của họ

    Search->>BehaviorLog: Ghi nhận query + category cuối cùng buyer chọn, phục vụ suy luận lần sau
```

---

# Sequence Diagram — Enhance: Change Product Category

Đây là **enhance**, flow hoàn toàn mới: khi người bán đổi category của sản phẩm sau khi đã đăng, index phải cập nhật đồng thời danh mục lẫn bộ facet thuộc tính tương ứng, đảm bảo sản phẩm không còn xuất hiện ở facet của category cũ.

```mermaid
sequenceDiagram
    actor Seller
    participant ProductSvc as Product Service
    participant DB as Product Database
    participant SchemaStore as Category Facet Schema
    participant Indexer as Unified Index Worker
    participant Index as Unified Product Index

    Seller->>ProductSvc: Đổi category sản phẩm (electronics -> furniture)
    ProductSvc->>DB: UPDATE products SET category_id = furniture
    DB-->>ProductSvc: Updated
    ProductSvc->>Indexer: Sync thay đổi category

    Indexer->>Index: Xóa toàn bộ PRODUCT_FACET_VALUE cũ theo schema electronics
    Index-->>Indexer: Facet cũ đã gỡ, sản phẩm không còn khớp bộ lọc electronics

    Indexer->>SchemaStore: Lấy facet schema mới của category furniture
    SchemaStore-->>Indexer: Danh sách facet_key hợp lệ cho furniture (chất liệu, kích thước...)

    Indexer->>Index: Cập nhật category_id trên PRODUCT_INDEX_DOCUMENT, ghi PRODUCT_FACET_VALUE mới theo schema furniture
    Index-->>Indexer: Indexed

    Note over Index: Việc xóa facet cũ và ghi facet mới xảy ra trong cùng 1 lần sync, không để sản phẩm lơ lửng ở cả 2 category cùng lúc
```
