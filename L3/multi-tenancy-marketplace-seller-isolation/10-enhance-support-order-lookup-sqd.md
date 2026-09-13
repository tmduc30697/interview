# Enhance sequence — Nhân viên chăm sóc khách hàng tra cứu đơn hàng (phạm vi giới hạn, có log)

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base không có flow riêng cho nhân viên chăm sóc khách hàng, ngụ ý trước đó không có ranh giới rõ giữa quyền hỗ trợ và toàn bộ dữ liệu kinh doanh seller. Đáp ứng yêu cầu 5 của đề bài: khi tra cứu một đơn hàng cụ thể để hỗ trợ khách mua, nhân viên chỉ thấy đúng phạm vi đơn hàng đó, không có quyền mặc định vào toàn bộ dữ liệu kinh doanh của seller liên quan, và mọi lần tra cứu đều được ghi log.

```mermaid
sequenceDiagram
    actor Support as Nhân viên chăm sóc khách hàng
    participant App as Customer Support Service
    participant Log as SUPPORT_ORDER_LOOKUP_LOG
    participant DB as ORDER / ORDER_ITEM store

    Support->>App: Tra cứu đơn hàng O theo yêu cầu hỗ trợ của khách mua
    App->>Log: Ghi SUPPORT_ORDER_LOOKUP_LOG(support_staff_id, order_id=O, reason, looked_up_at)
    App->>DB: SELECT * FROM order_item WHERE order_id = O
    DB-->>App: Toàn bộ ORDER_ITEM thuộc đơn O (có thể gồm nhiều seller, cần thiết để hỗ trợ khách mua)
    App-->>Support: Hiển thị đúng phạm vi đơn hàng O

    Support->>App: Thử xem thêm các đơn hàng khác hoặc số liệu doanh thu tổng của seller liên quan
    App->>App: Kiểm tra quyền, support role không có scope ngoài đơn hàng đang xử lý
    App-->>Support: Từ chối, không có quyền mặc định vào toàn bộ dữ liệu kinh doanh của seller
    Note over Log: Mọi lần tra cứu đều để lại log tra soát được, kể cả các lần bị từ chối mở rộng phạm vi
```
