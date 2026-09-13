# Nền tảng bất động sản tìm kiếm theo bản đồ và autocomplete địa chỉ

**Hệ thống:** Một nền tảng đăng tin bất động sản cho phép người tìm mua/thuê tra cứu theo khu vực trên bản đồ kết hợp autocomplete gợi ý địa chỉ/khu vực.

**Vai trò của flow:** Lập chỉ mục tin bất động sản kèm dữ liệu địa lý (tọa độ, ranh giới khu vực), phục vụ tìm kiếm theo vùng bản đồ và autocomplete địa chỉ chính xác.

**Yêu cầu cụ thể:**
- Tìm kiếm theo vùng hiển thị trên bản đồ (bounding box/bán kính từ một điểm) phải trả đúng và đủ các tin nằm trong vùng đó, xử lý chính xác các trường hợp tin ở biên giới vùng (gần đúng ranh giới quận/khu vực).
- Autocomplete địa chỉ phải xử lý được cách viết địa chỉ không thống nhất (viết tắt, thiếu dấu, thứ tự phường/quận/thành phố khác nhau) và gợi ý đúng địa chỉ chuẩn hóa, không yêu cầu người dùng gõ chính xác định dạng địa chỉ đầy đủ.
- Khi người đăng tin cập nhật tọa độ/vị trí của bất động sản (ví dụ sửa lại vị trí đánh dấu trên bản đồ cho chính xác hơn), index phải cập nhật ngay để tin xuất hiện đúng ở kết quả tìm kiếm theo vùng bản đồ mới, biến mất khỏi vùng cũ.
- Tin đã cho thuê/bán xong phải được gỡ khỏi kết quả tìm kiếm theo bản đồ ngay lập tức, tránh người tìm liên hệ vào tin không còn hiệu lực gây trải nghiệm xấu.
- Khi người dùng zoom/pan bản đồ liên tục để khám sát khu vực, hệ thống phải trả kết quả tìm kiếm theo vùng mới với độ trễ thấp cho từng lần thay đổi vùng nhìn, không tính toán lại toàn bộ từ đầu mỗi lần một cách tốn kém tài nguyên.

---

# ERD — Base (trước khi có tìm kiếm theo bản đồ)

Đây là **base**: mô hình dữ liệu suy luận cho nền tảng bất động sản *trước khi* có tìm kiếm theo bản đồ/autocomplete địa chỉ. Đề bài mô tả toàn bộ khả năng bản đồ/autocomplete là nội dung của enhance, nên base được suy luận là trạng thái trước đó: tin đăng chỉ có địa chỉ dạng text tự do, tìm kiếm theo bộ lọc cơ bản (giá, loại tin), chưa có tọa độ hay index địa lý.

```mermaid
erDiagram
    AGENT ||--o{ LISTING : posts

    AGENT {
        string agent_id PK
        string full_name
        string phone
    }

    LISTING {
        string listing_id PK
        string agent_id FK
        string address_text
        decimal price
        string listing_type
        string status
        datetime created_at
    }
```

---

# Sequence Diagram — Base: Search Listing

Đây là **base**, flow "tìm tin bất động sản" bằng bộ lọc cơ bản (giá, loại tin, khớp chuỗi địa chỉ text) — tiền đề cho thấy hạn chế: không có khả năng tìm theo vùng bản đồ hay autocomplete địa chỉ chuẩn hóa, phải gõ đúng định dạng địa chỉ mới ra kết quả.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Real Estate App
    participant DB as Listings Database

    Buyer->>WebApp: Nhập địa chỉ/khu vực dạng text, chọn khoảng giá
    WebApp->>DB: SELECT * FROM listings WHERE address_text LIKE %input% AND price BETWEEN min AND max
    DB-->>WebApp: Kết quả khớp chuỗi text (có thể bỏ sót do viết tắt/thiếu dấu)
    WebApp-->>Buyer: Hiển thị danh sách tin, không có bản đồ trực quan

    Note over WebApp,DB: Không có tọa độ, không tìm được theo vùng hiển thị trên bản đồ
```

---

# ERD — Enhance (sau khi có tìm kiếm theo bản đồ)

Đây là **enhance**: `LISTING` được bổ sung tọa độ `lat`/`lng`, thêm `LISTING_GEO_INDEX_DOCUMENT` phục vụ query vùng bản đồ hiệu quả, `ADDRESS_INDEX_ENTRY` phục vụ autocomplete địa chỉ chuẩn hóa, và `VIEWPORT_QUERY_CACHE` để trả kết quả nhanh khi zoom/pan liên tục. So với base, tìm kiếm không còn dựa trên `address_text` khớp chuỗi mà dựa trên tọa độ và index địa lý chuyên dụng.

```mermaid
erDiagram
    AGENT ||--o{ LISTING : posts
    LISTING ||--o| LISTING_GEO_INDEX_DOCUMENT : "indexed with coordinates"
    ADDRESS_INDEX_ENTRY ||--o{ LISTING : "resolves to"

    AGENT {
        string agent_id PK
        string full_name
        string phone
    }

    LISTING {
        string listing_id PK
        string agent_id FK
        string address_text
        decimal lat
        decimal lng
        decimal price
        string listing_type
        string status
        datetime created_at
    }

    LISTING_GEO_INDEX_DOCUMENT {
        string listing_id PK
        decimal lat
        decimal lng
        string geohash
        decimal price
        string status
        datetime indexed_at
    }

    ADDRESS_INDEX_ENTRY {
        string address_entry_id PK
        string normalized_address
        string alias_text
        decimal lat
        decimal lng
    }

    VIEWPORT_QUERY_CACHE {
        string cache_key PK
        string bounding_box
        string cached_result_ids
        datetime expires_at
    }
```

---

# Sequence Diagram — Enhance: Search by Map Region

Đây là **enhance**, flow hoàn toàn mới: tìm kiếm theo vùng hiển thị trên bản đồ (bounding box/bán kính), trả đúng và đủ tin trong vùng kể cả các tin ở gần đúng ranh giới quận/khu vực.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Real Estate App
    participant Search as Geo Search Service
    participant Index as Listing Geo Index

    Buyer->>WebApp: Kéo/xác định vùng xem trên bản đồ (bounding box)
    WebApp->>Search: Search request (bounding_box hoặc center + radius)

    Search->>Index: Query theo geohash/tọa độ nằm trong bounding box, nới biên nhẹ cho tin sát ranh giới
    Index-->>Search: Danh sách listing trong vùng, bao gồm tin gần biên

    Search->>Search: Lọc chính xác lại theo khoảng cách thực tế cho các tin ở biên
    Search-->>WebApp: Danh sách tin trong vùng bản đồ
    WebApp-->>Buyer: Hiển thị marker trên bản đồ
```

---

# Sequence Diagram — Enhance: Autocomplete Address

Đây là **enhance**, flow hoàn toàn mới: autocomplete xử lý địa chỉ viết tắt/thiếu dấu/thứ tự phường-quận-thành phố khác nhau, gợi ý đúng địa chỉ chuẩn hóa mà không cần người dùng gõ chính xác định dạng đầy đủ.

```mermaid
sequenceDiagram
    actor Buyer
    participant WebApp as Real Estate App
    participant Autocomplete as Address Autocomplete Service
    participant Index as Address Index

    Buyer->>WebApp: Gõ địa chỉ không thống nhất (viết tắt, thiếu dấu, sai thứ tự)
    WebApp->>Autocomplete: Autocomplete request (partial input)

    Autocomplete->>Autocomplete: Chuẩn hóa input (bỏ dấu, mở rộng viết tắt, không phụ thuộc thứ tự token)
    Autocomplete->>Index: Query trên alias_text/normalized_address đã chuẩn hóa tương tự
    Index-->>Autocomplete: Danh sách địa chỉ chuẩn hóa khớp, kèm tọa độ

    Autocomplete-->>WebApp: Gợi ý địa chỉ chuẩn hóa
    WebApp-->>Buyer: Hiển thị gợi ý, buyer chọn 1 địa chỉ đúng ý
```

---

# Sequence Diagram — Enhance: Update Listing Location

Đây là **enhance**, flow hoàn toàn mới: khi người đăng tin sửa lại tọa độ/vị trí trên bản đồ cho chính xác hơn, index phải cập nhật ngay để tin xuất hiện đúng vùng mới và biến mất khỏi vùng cũ.

```mermaid
sequenceDiagram
    actor Agent
    participant WebApp as Real Estate App
    participant DB as Listings Database
    participant Indexer as Geo Index Sync Worker
    participant Index as Listing Geo Index

    Agent->>WebApp: Kéo lại vị trí đánh dấu trên bản đồ cho chính xác hơn
    WebApp->>DB: UPDATE listings SET lat, lng = vị trí mới
    DB-->>WebApp: Updated
    WebApp->>Indexer: Trigger sync ngay khi tọa độ đổi

    Indexer->>Index: Xóa entry cũ theo geohash cũ
    Indexer->>Index: Ghi entry mới theo geohash của tọa độ mới
    Index-->>Indexer: Indexed

    Note over Index: Tin biến mất khỏi kết quả vùng bản đồ cũ và xuất hiện đúng ở vùng mới ngay lập tức
    WebApp-->>Agent: Vị trí đã cập nhật
```

---

# Sequence Diagram — Enhance: Mark Listing Unavailable

Đây là **enhance**, flow hoàn toàn mới: tin đã cho thuê/bán xong phải bị gỡ khỏi kết quả tìm kiếm theo bản đồ ngay lập tức, tránh người tìm liên hệ vào tin không còn hiệu lực.

```mermaid
sequenceDiagram
    actor Agent
    participant WebApp as Real Estate App
    participant DB as Listings Database
    participant Indexer as Geo Index Sync Worker
    participant Index as Listing Geo Index
    actor Buyer

    Agent->>WebApp: Đánh dấu tin đã cho thuê/bán xong
    WebApp->>DB: UPDATE listings SET status = unavailable
    DB-->>WebApp: Updated
    WebApp->>Indexer: Trigger sync ngay lập tức, priority cao

    Indexer->>Index: UPDATE listing_geo_index_document SET status = unavailable
    Index-->>Indexer: Indexed

    Buyer->>Index: Tìm kiếm theo vùng bản đồ
    Index-->>Buyer: Tin đã gỡ không còn xuất hiện trong kết quả

    WebApp-->>Agent: Tin đã được gỡ khỏi tìm kiếm
```
