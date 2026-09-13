# Enhance sequence — Configure IdP

Đây là **enhance**, cùng flow "Admin cấu hình lại IdP" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 2 của đề bài: không còn ghi đè trực tiếp `IDP_CONFIG`, mà phải qua `SSO_CHANGE_REQUEST` với xác minh danh tính chặt chẽ + duyệt nội bộ, để tránh kẻ tấn công giả danh admin trỏ SSO sang IdP do chúng kiểm soát.

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Console as SaaS Admin Console
    participant Verify as Identity Verification Service
    participant Security as Internal Security Review
    participant DB as SSO_CHANGE_REQUEST / IDP_CONFIG store

    Admin->>Console: Yêu cầu đổi/cấu hình lại IdP, nộp metadata IdP mới
    Console->>Verify: Xác minh danh tính admin (out-of-band: xác nhận qua contact đã đăng ký của tổ chức, không chỉ session hiện tại)
    Verify-->>Console: Kết quả xác minh
    alt Xác minh thất bại hoặc đáng ngờ
        Console-->>Admin: Từ chối, yêu cầu xác minh lại/qua kênh khác
    else Xác minh hợp lệ
        Console->>DB: Tạo SSO_CHANGE_REQUEST (type=planned_cutover, verification_status=verified)
        DB->>Security: Đưa vào hàng chờ duyệt nội bộ
        Security-->>DB: Duyệt request
        DB->>DB: Tạo IDP_CONFIG mới với status=candidate
        DB-->>Console: Request approved, sẵn sàng cho bước cutover
        Console-->>Admin: Thông báo request đã duyệt
    end
```
