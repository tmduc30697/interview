# Enhance sequence - Keyset pagination cho phân trang sâu

Đây là trạng thái **enhance**, mô tả 1 flow hoàn toàn mới phát sinh từ đề bài: chưa từng tồn tại ở base (base chỉ có OFFSET pagination). Flow này được chọn vì đề bài yêu cầu rõ chuyển từ OFFSET sang keyset pagination (dựa trên giá trị cột index cuối cùng đã thấy) để giữ độ trễ ổn định khi user phân trang sâu (page 200+), thay vì để DB quét và bỏ qua toàn bộ dòng phía trước.

```mermaid
sequenceDiagram
    actor User
    participant WebApp as Storefront Web/App
    participant CatalogAPI as Catalog Service
    participant DB as Product DB (Postgres)

    User->>WebApp: Đang ở trang 200, nhấn "Trang sau", sort theo mới nhất
    WebApp->>CatalogAPI: GET /products?category=..&sort=newest&after_created_at=2026-08-01T10:00:00&after_id=98213
    Note over CatalogAPI: after_created_at/after_id là cursor lấy từ dòng cuối cùng của trang trước, không phải số trang
    CatalogAPI->>DB: SELECT id,name,created_at FROM products WHERE category_id=? AND (created_at,id) < (?,?) ORDER BY created_at DESC, id DESC LIMIT 20
    Note over DB: Dùng idx_cat_created_at, seek trực tiếp tới vị trí cursor bằng index, không cần quét bỏ qua các dòng phía trước
    DB-->>CatalogAPI: 20 rows kèm created_at/id của dòng cuối cùng
    CatalogAPI-->>WebApp: Danh sách trang tiếp theo, next_cursor = created_at/id dòng cuối
    WebApp-->>User: Hiển thị kết quả, độ trễ không đổi dù đang ở trang sâu bao nhiêu
```

Khác biệt cốt lõi so với OFFSET ở base: DB không còn phải đếm và bỏ qua N dòng đầu, mà seek thẳng tới vị trí trong index nhờ điều kiện `(created_at, id) < (cursor_created_at, cursor_id)`, nên độ trễ ổn định bất kể phân trang sâu tới đâu.
