# Enhance sequence — Xem dashboard người bán (enforce seller_id ở tầng data layer)

Đây là **enhance**, cùng flow "Xem dashboard người bán" đã có ở base nhưng nay thay đổi theo yêu cầu 1 của đề bài: `seller_id` được enforce ngay tại tầng data/query layer — kể cả Internal Aggregate API dùng chung cho nhiều mục đích khác của nền tảng — thay vì chỉ lọc ở UI như base.

```mermaid
sequenceDiagram
    actor Seller as Seller A
    participant App as Seller Dashboard App
    participant Agg as Internal Aggregate API (dùng chung nhiều mục đích)
    participant Scope as Query Scoping Layer (bắt buộc mọi caller)
    participant DB as ORDER_ITEM store

    Seller->>App: Xem đơn hàng/doanh thu/tồn kho của mình
    App->>Agg: Gọi API tổng hợp số liệu, kèm seller_id=A trong security context
    Agg->>Scope: Chuyển query xuống, bắt buộc đi qua Query Scoping Layer
    Scope->>Scope: Tiêm điều kiện WHERE seller_id = A vào mọi query, không cho phép bỏ qua
    Scope->>DB: Query ORDER_ITEM đã bị giới hạn seller_id = A ngay tại tầng data
    DB-->>Scope: Chỉ trả ORDER_ITEM của Seller A
    Scope-->>Agg: Dữ liệu đã an toàn
    Agg-->>App: Trả về đúng phạm vi Seller A
    App-->>Seller: Hiển thị dashboard

    Note over Scope: Mọi caller khác của Internal Aggregate API (vd tool phân tích nội bộ, job vận hành) cũng đi qua cùng Query Scoping Layer, muốn truy vấn xuyên seller phải có context đặc quyền riêng, tự nó cũng bị audit
```
