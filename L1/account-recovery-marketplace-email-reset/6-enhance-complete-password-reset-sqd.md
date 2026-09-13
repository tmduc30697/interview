# Enhance sequence — Complete password reset

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tách riêng ở base (ở base, bước "đặt mật khẩu mới" chỉ là phần cuối gộp trong 1 flow duy nhất, không có các kiểm soát này). Nối tiếp flow "Forgot password" sau khi user bấm vào link, đáp ứng yêu cầu 2 và 4 của đề bài: token chỉ dùng 1 lần + hết hạn ngắn, và sau khi đổi mật khẩu thành công thì đăng xuất mọi session khác + gửi email thông báo.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant DB as PASSWORD_RESET_TOKEN / USER store
    participant SessionStore as SESSION store
    participant Email as Email Gateway

    User->>App: Mở link reset (kèm token), nhập mật khẩu mới
    App->>DB: Tra PASSWORD_RESET_TOKEN theo token_hash
    alt Token không tồn tại, đã dùng (status=used), đã bị vô hiệu (status=invalidated), hoặc hết hạn
        DB-->>App: Token không hợp lệ
        App-->>User: "Link không hợp lệ hoặc đã hết hạn", yêu cầu gửi lại link mới
    else Token active và chưa hết hạn
        DB-->>App: Token hợp lệ
        App->>DB: Cập nhật password_hash của USER
        App->>DB: Đặt token status=used, used_at=now
        App->>SessionStore: Tìm toàn bộ SESSION đang active của user
        SessionStore->>SessionStore: Đặt revoked_at cho mọi session khác (trừ thiết bị vừa thực hiện đổi)
        App->>SessionStore: Tạo/giữ session cho thiết bị hiện tại
        App->>Email: Gửi email "mật khẩu của bạn vừa được đổi" kèm hướng dẫn báo cáo nếu không phải chính chủ
        App-->>User: Đổi mật khẩu thành công, tiếp tục dùng trên thiết bị này
    end
```
