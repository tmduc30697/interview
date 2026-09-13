# Enhance sequence — Planned IdP cutover

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 4 của đề bài: khi tổ chức chủ động đổi IdP theo kế hoạch, cho phép chạy song song IdP cũ và mới trong 1 khoảng thời gian chuyển tiếp, để không có khoảnh khắc nào cả tổ chức bị khóa. Flow này tiếp nối ngay sau khi `SSO_CHANGE_REQUEST` được duyệt ở flow "Configure IdP".

```mermaid
sequenceDiagram
    actor Admin as Org Admin
    participant Console as SaaS Admin Console
    participant DB as IDP_CONFIG store
    participant Monitor as Migration Monitor
    actor Employees

    Console->>DB: Kích hoạt IDP_CONFIG mới (status=active), IDP_CONFIG cũ chuyển status=retiring
    Note over DB: Cả 2 config cùng hiệu lực trong cutover window (valid_from/valid_until)
    Employees->>DB: Đăng nhập qua IdP cũ hoặc mới (xem flow SSO Login sau enhance)
    Monitor->>DB: Theo dõi tỉ lệ đăng nhập thành công theo từng IDP_CONFIG
    Monitor-->>Admin: Báo cáo tiến độ migrate nhân viên sang IdP mới
    Admin->>Console: Xác nhận đủ điều kiện kết thúc cutover (đa số đã chuyển, hoặc hết hạn window)
    Console->>DB: Đặt IDP_CONFIG cũ status=retired
    Console->>DB: Cập nhật SSO_CHANGE_REQUEST status=completed
    DB-->>Admin: Cutover hoàn tất, chỉ còn IdP mới active
```
