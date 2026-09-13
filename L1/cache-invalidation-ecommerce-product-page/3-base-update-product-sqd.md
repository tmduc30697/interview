# Base sequence — Update product (chỉ dựa TTL, không invalidate)

Đây là **base**, flow "Admin đổi giá/tồn kho" ở trạng thái hiện tại — cập nhật DB xong là kết thúc, không đụng gì tới cache. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng lỗ hổng "chờ TTL" ở đây.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminApp as Admin Panel
    participant DB as PRODUCT store

    Admin->>AdminApp: Đổi giá hoặc tồn kho sản phẩm
    AdminApp->>DB: Cập nhật PRODUCT (price/stock, updated_at)
    DB-->>AdminApp: Cập nhật thành công
    AdminApp-->>Admin: "Cập nhật thành công"
    Note over DB: Không có bước invalidate cache nào được gọi — PRODUCT_CACHE_ENTRY vẫn giữ giá trị cũ cho tới khi ttl_seconds=3600 tự hết hạn, khách hàng có thể thấy giá sai tới 1 giờ
```
