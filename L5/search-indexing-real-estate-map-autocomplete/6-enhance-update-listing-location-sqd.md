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
