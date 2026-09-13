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
