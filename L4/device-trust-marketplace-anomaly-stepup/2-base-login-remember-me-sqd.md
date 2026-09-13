# Base sequence — Login remember me (session dài hạn, không phân biệt hành động sau này)

Đây là **base**, flow đăng nhập với tuỳ chọn "remember me" ở trạng thái hiện tại: sau khi xác thực, hệ thống cấp 1 session dài hạn (vd 30 ngày), session này được dùng cho mọi hành động sau đó mà không phân biệt mức độ nhạy cảm. Đây là nền để so sánh với yêu cầu 1 của đề bài (cần tách biệt session dài hạn khỏi các hành động nhạy cảm).

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant Auth as Auth Service
    participant DB as Database

    User->>App: Đăng nhập, tick chọn "Ghi nhớ đăng nhập"
    App->>Auth: Xác thực username/password
    Auth-->>App: Xác thực thành công

    App->>DB: INSERT SESSION (remember_me=true, expires_at=now+30 ngày, ip_address, user_agent)
    DB-->>App: Session token dài hạn được tạo

    App-->>User: Đăng nhập thành công, phiên có hiệu lực 30 ngày
    Note over App,DB: Cùng 1 session token này sẽ được dùng cho mọi hành động sau đó, kể cả xem sản phẩm, đổi địa chỉ, đổi phương thức thanh toán, hay thanh toán
```
