# Enhance sequence — Auto-logout theo idle timeout cho máy dùng chung

Đây là **enhance**, flow hoàn toàn mới, dành cho máy dùng chung (quầy giao dịch, máy tính công cộng). sessionStorage tự xoá khi đóng tab là chưa đủ vì nhân viên sau có thể dùng lại tab đang mở sẵn; hệ thống cần chủ động đếm thời gian không thao tác và tự đăng xuất, xoá rõ ràng toàn bộ cache/state, không phụ thuộc vào việc user có đóng tab hay không. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant Idle as IDLE_TIMEOUT_STATE
    participant API as Auth API
    participant Mem as ACCESS_TOKEN_MEMORY
    participant Storage as sessionStorage (SESSION_UI_STATE)

    User->>App: Thao tác (click, gõ phím, cuộn trang)
    App->>Idle: Cập nhật last_activity_at=now mỗi khi có thao tác

    loop Kiểm tra định kỳ
        App->>Idle: So sánh now - last_activity_at với idle_timeout_ms
        alt Sắp hết hạn (còn ví dụ 30 giây)
            App-->>User: Hiển thị cảnh báo sắp tự đăng xuất do không thao tác
        else Chưa tới ngưỡng
            App->>App: Không làm gì thêm
        end
    end

    alt Vượt ngưỡng idle_timeout_ms mà không có thao tác nào
        App->>API: Gọi logout tự động (revoke session, tương tự flow cross-tab-logout)
        App->>Mem: Xoá access_token khỏi bộ nhớ
        App->>Storage: Xoá sạch sessionStorage của tab này
        App-->>User: Chuyển về màn hình đăng nhập, ghi rõ lý do tự động đăng xuất do không thao tác
    end
```
