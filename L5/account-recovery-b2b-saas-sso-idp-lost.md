# SaaS B2B khôi phục quyền truy cập khi công ty khách hàng đổi hoặc mất Identity Provider

**Hệ thống:** Một SaaS doanh nghiệp mà phần lớn tài khoản đăng nhập qua SSO của công ty khách hàng (Okta/Azure AD...), không có mật khẩu riêng trên nền tảng.

**Vai trò của flow:** Xử lý tình huống công ty khách hàng đổi nhà cung cấp IdP, mất quyền quản trị IdP cũ, hoặc IdP cũ ngừng hoạt động — khiến toàn bộ nhân viên công ty đó không thể đăng nhập qua SSO như bình thường, cần một quy trình khôi phục quyền truy cập không phụ thuộc vào chính IdP đã hỏng.

**Yêu cầu cụ thể:**
- Vì tài khoản SSO-only không có mật khẩu, khi IdP của cả một tổ chức khách hàng ngừng hoạt động, toàn bộ nhân viên công ty đó bị khóa cùng lúc — cần một cơ chế khôi phục cấp tổ chức, không phải xử lý từng user riêng lẻ, để tránh hàng trăm ticket hỗ trợ cùng lúc và tránh gián đoạn nghiệp vụ kéo dài.
- Việc xác minh ai có quyền yêu cầu chuyển đổi hoặc cấu hình lại SSO cho một tổ chức phải chặt chẽ, để tránh kẻ tấn công giả danh admin công ty khách hàng nhằm chiếm quyền truy cập toàn bộ dữ liệu tổ chức đó thông qua việc trỏ SSO sang một IdP do chúng kiểm soát.
- Cần thiết lập sẵn từ trước một tài khoản break-glass cấp tổ chức cho ít nhất một admin của mỗi tổ chức khách hàng lớn, không phải tạo mới lúc khẩn cấp, để có đường vào dự phòng khi SSO chính hỏng hoàn toàn, kèm quy trình giám sát chặt việc sử dụng tài khoản này.
- Khi công ty khách hàng chủ động đổi IdP theo kế hoạch (không phải sự cố), quy trình cutover phải cho phép chạy song song cả IdP cũ và mới trong một khoảng thời gian chuyển tiếp, để tránh có khoảnh khắc không IdP nào hoạt động khiến toàn bộ nhân viên bị khóa giữa chừng.
- Phải phân biệt rõ giữa lỗi tạm thời của IdP (downtime ngắn, sự cố mạng) và việc tổ chức thực sự cần chuyển IdP, tránh kích hoạt quy trình khôi phục khẩn cấp cấp tổ chức — vốn có rủi ro bảo mật cao hơn — chỉ vì một sự cố ngắn hạn có thể tự phục hồi.
