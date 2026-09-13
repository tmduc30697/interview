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
