# Enhance sequence — Logout đồng bộ mọi tab đang mở

Đây là **enhance**, cùng flow "logout" như ở base nhưng thay đổi ở chỗ: đăng xuất ở 1 tab phải làm mất hiệu lực token ở toàn bộ tab đang mở, không chỉ tab bấm logout, và dữ liệu phi nhạy cảm trong sessionStorage cũng phải được xoá sạch ở mọi tab. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor User as Người dùng (Tab A, bấm logout)
    participant TabA as Tab A
    participant API as Auth API
    participant DB as SERVER_SESSION
    participant Channel as AUTH_BROADCAST_EVENT (kênh auth riêng)
    participant TabB as Tab B (cùng tài khoản)

    User->>TabA: Bấm đăng xuất
    TabA->>API: POST logout
    API->>DB: Đánh dấu SERVER_SESSION.revoked_at=now, thu hồi refresh token
    API-->>TabA: Xác nhận đã revoke, đồng thời xoá cookie refresh_token
    TabA->>TabA: Xoá ACCESS_TOKEN_MEMORY và SESSION_UI_STATE của chính tab A
    TabA->>Channel: Phát AUTH_BROADCAST_EVENT(event_type=logout, origin_tab_id=A)

    Channel-->>TabB: Nhận sự kiện logout
    TabB->>TabB: Xoá ACCESS_TOKEN_MEMORY và SESSION_UI_STATE của tab B
    TabB-->>User: Chuyển Tab B về màn hình đăng nhập ngay, dù không ai bấm logout ở tab này

    Note over TabB,API: Nếu Tab B cố gọi API bằng access_token cũ trước khi kịp nhận broadcast, server vẫn từ chối vì refresh token cho session đó đã bị revoke
```
