# Enhance sequence — Xem tài liệu (3 loại tài nguyên, tách quyền vận hành khỏi quyền đọc nội dung)

Đây là **enhance**, cùng flow "Xem tài liệu" đã có ở base nhưng nay thay đổi theo yêu cầu 1 và 3 của đề bài: không còn 1 quy tắc cứng nhắc duy nhất mà rẽ nhánh theo `resource_type` (cách ly tuyệt đối / dùng chung toàn công ty / chia sẻ liên phòng ban), đồng thời admin IT mặc định chỉ có quyền vận hành, muốn đọc nội dung nghiệp vụ phải có `BUSINESS_CONTENT_GRANT` tường minh.

```mermaid
sequenceDiagram
    actor Emp as Nhân viên (phòng kinh doanh)
    actor ItAdmin as Admin IT
    participant App as Internal Tool Service
    participant DB as DOCUMENT store
    participant Share as CROSS_DEPARTMENT_SHARE
    participant Grant as BUSINESS_CONTENT_GRANT

    Emp->>App: Yêu cầu xem tài liệu D
    App->>DB: Lấy DOCUMENT(D).resource_type, department_id
    DB-->>App: resource_type, department_id
    alt resource_type = company_wide
        App-->>Emp: Cho phép xem, không cần kiểm tra phòng ban
    else resource_type = department_only
        App->>App: So sánh department_id với employee.department_id hiện tại
        App-->>Emp: Cho phép nếu khớp, từ chối nếu không khớp
    else resource_type = cross_department_shared
        App->>Share: Kiểm tra có SHARE_TARGET_DEPARTMENT khớp phòng ban Emp, còn hiệu lực (valid_from <= now <= valid_until, status=active)
        Share-->>App: Kết quả kiểm tra
        App-->>Emp: Cho phép nếu còn hiệu lực, từ chối nếu hết hạn hoặc không nằm trong danh sách chia sẻ
    end

    ItAdmin->>App: Yêu cầu backup/cấu hình hệ thống
    App-->>ItAdmin: Cho phép, đây là quyền vận hành mặc định

    ItAdmin->>App: Yêu cầu xem nội dung tài liệu hồ sơ lương (phòng nhân sự)
    App->>Grant: Kiểm tra BUSINESS_CONTENT_GRANT(employee_id=ItAdmin, department_id=nhân sự, revoked_at=null)
    Grant-->>App: Không tìm thấy grant hợp lệ
    App-->>ItAdmin: Từ chối, quyền vận hành không mặc định bao gồm quyền đọc nội dung nghiệp vụ
```
