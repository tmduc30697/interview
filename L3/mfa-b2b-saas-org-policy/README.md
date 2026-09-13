# SaaS B2B cho phép admin công ty tự cấu hình chính sách MFA bắt buộc theo tổ chức

**Hệ thống:** Một SaaS quản lý dự án/CRM bán cho nhiều công ty khách hàng (multi-tenant), mỗi công ty có admin riêng quản lý chính sách bảo mật cho nhân viên của họ.

**Vai trò của flow:** Cho phép org admin tự cấu hình chính sách MFA bắt buộc (bật/tắt, phương thức được phép, áp dụng cho nhóm role nào) riêng cho tổ chức của họ, độc lập với các tenant khác trên cùng nền tảng.

**Yêu cầu cụ thể:**
- Khi admin bật chính sách bắt buộc MFA cho toàn tổ chức, những user đang có session hoạt động nhưng chưa enroll MFA không thể bị ép logout tức thì vì sẽ gây gián đoạn hàng loạt, nhưng cũng không thể được dùng vô thời hạn mà không enroll — cần một grace period có giới hạn rõ ràng kèm nhắc nhở tăng dần.
- Chính sách MFA có thể cấu hình khác nhau theo role trong cùng tổ chức (ví dụ chỉ bắt buộc cho admin/owner, không bắt buộc cho viewer) — khi một user được đổi role giữa chừng (từ viewer lên admin), chính sách MFA mới phải áp dụng ngay, không chờ tới lần đăng nhập kế tiếp mới phát hiện ra là chưa đủ điều kiện.
- Vì một tài khoản email có thể tham gia nhiều tổ chức trên cùng nền tảng với chính sách MFA khác nhau, cần định nghĩa rõ MFA áp dụng theo ngữ cảnh org nào trong phiên làm việc, tránh tình huống user né được yêu cầu MFA của tổ chức chặt hơn bằng cách chuyển sang truy cập qua context của tổ chức khác lỏng hơn.
- Khi admin hạ cấp hoặc tắt chính sách MFA bắt buộc (ví dụ do đổi nhà cung cấp SSO), đây là hành động làm giảm bảo mật toàn tổ chức nên cần yêu cầu chính admin đó xác thực MFA để thực hiện thay đổi, đồng thời ghi log và thông báo cho các admin khác trong tổ chức biết.
- Khi tổ chức bắt buộc một phương thức MFA cụ thể chặt hơn (ví dụ chỉ cho WebAuthn, cấm SMS OTP), những user đã enroll SMS theo chính sách cũ phải được yêu cầu enroll lại phương thức mới trong một mốc thời gian rõ ràng, không bị mất quyền truy cập đột ngột ngay khi chính sách vừa thay đổi.
