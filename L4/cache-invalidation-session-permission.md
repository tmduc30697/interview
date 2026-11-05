# Cache session/permission cho hệ thống phân quyền

**Hệ thống:** Một SaaS B2B cache thông tin quyền hạn (permission/role) của user để tránh query DB mỗi request, ảnh hưởng tới việc kiểm soát truy cập.

**Vai trò của flow:** Invalidate cache permission ngay khi quyền của user bị thay đổi (bị revoke quyền, đổi role), vì đây là dữ liệu bảo mật — cache cũ có thể dẫn tới lỗ hổng cho phép truy cập trái phép.

**Yêu cầu cụ thể:**
- Khi admin revoke quyền của một user, cache permission liên quan phải bị invalidate ngay lập tức và có xác nhận (không chấp nhận eventual consistency dạng "vài giây sau mới có hiệu lực" cho việc revoke quyền).
- Trong kiến trúc nhiều instance service cùng cache permission cục bộ, phải có cơ chế broadcast invalidation tới toàn bộ instance (ví dụ qua pub/sub) khi có thay đổi quyền, không chỉ invalidate ở một instance xử lý request đổi quyền.
- Nếu broadcast invalidation thất bại một phần (một instance không nhận được message do mất kết nối tạm thời), phải có cơ chế dự phòng (TTL rất ngắn cho cache permission, ví dụ vài giây) để giới hạn "cửa sổ rủi ro" tối đa.
- Có test case cụ thể: user đang có session hoạt động bị revoke quyền admin ngay khi đang dùng — request tiếp theo của họ (sau invalidation) phải bị chặn đúng theo quyền mới, không dùng permission cache cũ.
- Log đầy đủ mọi lần invalidate permission cache (ai, quyền gì, khi nào) để phục vụ audit an ninh khi cần điều tra truy cập trái phép.
