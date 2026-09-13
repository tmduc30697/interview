# Base sequence — Configure IdP

Đây là **base**, flow "Admin cấu hình lại IdP cho tổ chức" ở trạng thái hiện tại — chưa có xác minh chặt chẽ. Flow này liên quan mật thiết tới enhance vì yêu cầu thứ 2 của đề bài ("việc xác minh ai có quyền yêu cầu chuyển đổi/cấu hình lại SSO phải chặt chẽ") chính là siết lại flow này để tránh kẻ tấn công giả danh admin trỏ SSO sang IdP do chúng kiểm soát.

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Console as SaaS Admin Console
    participant DB as IDP_CONFIG store

    Admin->>Console: Đăng nhập (qua SSO hiện tại)
    Admin->>Console: Mở màn hình SSO Settings
    Admin->>Console: Nhập metadata IdP mới (entity ID, cert, endpoint)
    Console->>Console: Validate định dạng metadata
    Console->>DB: Ghi đè IDP_CONFIG đang active của org
    DB-->>Console: Cập nhật thành công
    Console-->>Admin: Xác nhận cấu hình mới có hiệu lực ngay
    Note over DB: Không có bước xác minh danh tính/approval bổ sung
```
