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
