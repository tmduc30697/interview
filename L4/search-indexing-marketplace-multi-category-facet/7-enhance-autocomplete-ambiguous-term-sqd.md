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
