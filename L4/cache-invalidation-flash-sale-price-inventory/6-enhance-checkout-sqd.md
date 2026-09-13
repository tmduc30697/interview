# Enhance sequence — Checkout (invalidate chủ động + double-check giá)

Đây là **enhance**, cùng flow "Checkout" đã có ở base nhưng nay thay đổi theo yêu cầu 1, 4 và 5 của đề bài: trừ kho xong invalidate ngay `INVENTORY_CACHE_ENTRY` thay vì chờ TTL, và backend đối chiếu lại giá thực tế tại thời điểm submit thay vì tin thẳng giá client gửi lên.

```mermaid
sequenceDiagram
    actor Customer
    participant App as E-commerce App
    participant DB as PRODUCT / ORDER store
    participant InvCache as INVENTORY_CACHE_ENTRY store
    participant Log as DISPLAY_ACCURACY_LOG store

    Customer->>App: Submit đơn hàng kèm giá đã thấy trên trang (displayed_price)
    App->>DB: Đọc authoritative_price hiện tại của sản phẩm (theo lịch sale tại thời điểm submit)
    DB-->>App: Trả authoritative_price
    App->>App: So sánh displayed_price với authoritative_price
    alt Khớp nhau
        App->>DB: Tạo ORDER với charged_price = authoritative_price
    else Không khớp (giá đã đổi giữa lúc xem trang và lúc submit)
        App->>Log: Ghi DISPLAY_ACCURACY_LOG (metric_type=price_mismatch, displayed_value, actual_value)
        App->>DB: Tạo ORDER với charged_price = authoritative_price (không tin giá cũ khách gửi lên)
        App-->>Customer: Thông báo giá đã cập nhật, xác nhận lại trước khi charge nếu chênh lệch lớn
    end
    App->>DB: Trừ stock trực tiếp trong PRODUCT
    DB-->>App: Trừ kho thành công
    App->>InvCache: Publish INVENTORY_INVALIDATION_EVENT → invalidate INVENTORY_CACHE_ENTRY(product_id) ngay lập tức
    InvCache-->>App: Đã invalidate (không chờ TTL vài giây tự hết hạn)
    App-->>Customer: Đặt hàng thành công với giá chính xác
```
