# Ngân hàng số bắt buộc MFA cho đăng nhập và giao dịch nhạy cảm

**Hệ thống:** Một app ngân hàng số cho phép chuyển tiền, thanh toán hóa đơn, xem sao kê.

**Vai trò của flow:** MFA bắt buộc ngay từ lần đăng nhập đầu tiên (TOTP app + SMS backup), và step-up MFA riêng cho các giao dịch có giá trị lớn hoặc thêm người thụ hưởng mới.

**Yêu cầu cụ thể:**
- Bắt buộc enroll ít nhất một phương thức MFA trước khi được sử dụng bất kỳ tính năng nào của app, không cho phép "bỏ qua để sau".
- Giao dịch vượt một ngưỡng số tiền hoặc thêm beneficiary mới phải yêu cầu xác thực MFA lại dù session đăng nhập còn hạn (step-up theo hành vi, không chỉ theo thời gian).
- Mã OTP (TOTP/SMS) chỉ dùng được một lần, hết hạn sau một khoảng thời gian ngắn, và phải giới hạn số lần thử sai trước khi tạm khóa xác thực.
- Nếu SMS gửi OTP thất bại hoặc chậm, phải có phương án fallback rõ ràng (ví dụ dùng TOTP app) mà không làm người dùng bị kẹt hoàn toàn ngoài tài khoản.
- Mọi thay đổi liên quan đến MFA (thêm/xóa thiết bị, đổi số điện thoại nhận OTP) phải tự nó yêu cầu xác thực MFA hiện tại, và gửi thông báo cảnh báo qua kênh khác (email) ngay khi thay đổi.
