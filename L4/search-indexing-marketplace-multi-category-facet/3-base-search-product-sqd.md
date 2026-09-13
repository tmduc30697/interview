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
