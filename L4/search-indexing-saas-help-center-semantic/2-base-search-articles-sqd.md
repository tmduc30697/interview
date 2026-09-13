# Sequence Diagram — Base: Search Articles

Đây là **base**, flow "tìm bài viết trợ giúp" bằng khớp từ khóa thuần túy — tiền đề cho thấy hạn chế: khách hỏi "không đăng nhập được" sẽ không tìm ra bài "khắc phục lỗi xác thực tài khoản" vì không có từ nào trùng khớp.

```mermaid
sequenceDiagram
    actor Customer
    participant HelpCenter as Help Center App
    participant DB as Articles Database

    Customer->>HelpCenter: Gõ "không đăng nhập được"
    HelpCenter->>DB: SELECT * FROM articles WHERE status = published AND (title LIKE %keyword% OR content LIKE %keyword%)
    DB-->>HelpCenter: Không khớp bài "khắc phục lỗi xác thực tài khoản" vì không trùng từ khóa
    HelpCenter-->>Customer: Không tìm thấy kết quả phù hợp, dù bài viết đúng đã tồn tại

    Note over HelpCenter,DB: Chỉ khớp chuỗi từ, không hiểu được diễn đạt khác từ ngữ nhưng cùng ý nghĩa
```
