# Sequence Diagram — Enhance: Search Articles

Đây là **enhance**, flow "tìm bài viết" đã tồn tại ở base ([2-base-search-articles-sqd.md](2-base-search-articles-sqd.md)) nay thay đổi cốt lõi: query qua embedding để xếp hạng theo ngữ nghĩa thay vì chỉ khớp từ khóa, đồng thời lọc đúng phiên bản theo gói sản phẩm khách hàng đang dùng, mà không làm chậm đáng kể thời gian trả kết quả.

```mermaid
sequenceDiagram
    actor Customer
    participant HelpCenter as Help Center App
    participant Search as Search Service
    participant Index as Article Search Index

    Customer->>HelpCenter: Gõ "không đăng nhập được" (product_plan của customer)
    HelpCenter->>Search: Search request (query, product_plan)

    Search->>Search: Encode query thành vector ngữ nghĩa
    Search->>Index: Query kết hợp semantic vector similarity + keyword, filter product_plan phù hợp
    Index-->>Search: Bài "khắc phục lỗi xác thực tài khoản" khớp cao dù không trùng từ khóa

    Search->>Search: Xếp hạng theo độ liên quan ngữ nghĩa, loại các version không thuộc plan của customer
    Search-->>HelpCenter: Kết quả đúng ý định người hỏi
    HelpCenter-->>Customer: Hiển thị bài viết phù hợp, đúng phiên bản họ có quyền xem
```
