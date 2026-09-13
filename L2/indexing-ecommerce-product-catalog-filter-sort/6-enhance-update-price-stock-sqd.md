# Enhance sequence - Update giá và tồn kho với overhead index mới

Đây là trạng thái **enhance** của flow update-price-stock. So với base, flow này thay đổi ở chỗ: mỗi lần ghi giờ phải maintain thêm các composite/partial index mới tạo (`idx_cat_brand_price`, `idx_cat_created_at`, `idx_cat_sold_partial`), đúng với đánh đổi mà đề bài nêu ra giữa số lượng index phục vụ đọc và overhead ghi trên bảng products được cập nhật real-time. Partial index được thiết kế để giảm bớt overhead này bằng cách chỉ chứa các dòng còn hàng.

```mermaid
sequenceDiagram
    actor Admin as Admin/Pricing Job
    participant PricingSvc as Pricing/Inventory Service
    participant DB as Product DB (Postgres)

    Admin->>PricingSvc: Cập nhật giá hoặc trừ tồn kho sau khi có đơn hàng
    PricingSvc->>DB: UPDATE products SET price=?, stock_qty=?, updated_at=now() WHERE id=?
    Note over DB: Phải maintain thêm idx_cat_brand_price vì price/category/brand nằm trong index này
    Note over DB: Phải maintain thêm idx_cat_created_at nếu created_at hoặc category_id đổi, thường không đổi nên ít tốn kém hơn
    alt stock_qty chuyển từ 0 sang dương, hoặc từ dương về 0
        DB->>DB: Thêm hoặc xóa entry trong idx_cat_sold_partial, vì partial index chỉ chứa dòng thoả predicate stock_qty > 0
    else stock_qty vẫn giữ nguyên trạng thái còn hàng hoặc hết hàng
        DB->>DB: Không đổi entry trong idx_cat_sold_partial, giảm overhead so với index đầy đủ trên toàn bộ bảng
    end
    DB-->>PricingSvc: OK
    PricingSvc-->>Admin: Xác nhận cập nhật
```

So với base, chi phí ghi tăng lên do phải maintain 2-3 composite/partial index thay vì chỉ PK, nhưng đây là đánh đổi có chủ đích: chỉ giữ vài index phổ biến nhất (dựa trên `FILTER_USAGE_STAT`) thay vì tạo index cho mọi tổ hợp filter có thể có.
