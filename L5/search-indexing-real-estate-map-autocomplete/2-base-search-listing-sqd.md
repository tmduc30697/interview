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
