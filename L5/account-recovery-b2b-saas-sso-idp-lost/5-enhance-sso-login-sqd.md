# Enhance sequence — SSO Login

Đây là **enhance**, cùng flow "SSO Login" đã có ở base nhưng nay thay đổi để hỗ trợ yêu cầu "chạy song song IdP cũ và mới trong thời gian chuyển tiếp": app không còn giả định org chỉ có đúng 1 `IDP_CONFIG` active, mà phải khớp assertion trả về với đúng cấu hình (cũ hoặc mới) đang hiệu lực tại thời điểm đó — nhờ vậy không có khoảnh khắc nào cả tổ chức bị khóa giữa chừng khi cutover có kế hoạch.

```mermaid
sequenceDiagram
    actor Employee
    participant App as SaaS App (SP)
    participant DB as IDP_CONFIG store
    participant IdP as Old hoặc New IdP

    Employee->>App: Truy cập app bằng email công ty
    App->>DB: Tìm toàn bộ IDP_CONFIG có status active/retiring của org
    DB-->>App: Trả về 1 hoặc 2 config (song song trong lúc cutover)
    App-->>Employee: Redirect sang IdP tương ứng (theo lựa chọn/home realm discovery)
    Employee->>IdP: Đăng nhập tại IdP (cũ hoặc mới)
    IdP-->>App: Trả assertion đã ký, kèm issuer
    App->>DB: Khớp issuer với đúng IDP_CONFIG (cũ hoặc mới) đang hiệu lực
    App->>App: Verify chữ ký assertion bằng cert của config vừa khớp
    App->>App: Tìm/khớp USER, tạo SESSION
    App-->>Employee: Đăng nhập thành công dù đang trong giai đoạn cutover
```
