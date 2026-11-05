# Marketplace thương mại điện tử với quên mật khẩu qua email

**Hệ thống:** Một sàn thương mại điện tử tiêu dùng phổ thông, tài khoản đăng ký bằng email/password.

**Vai trò của flow:** Luồng quên mật khẩu tiêu chuẩn — user nhập email, nhận link reset, đặt mật khẩu mới — phải an toàn ở quy mô lớn và chống được các kiểu tấn công phổ biến.

**Yêu cầu cụ thể:**
- Không được để endpoint "quên mật khẩu" lộ thông tin email có tồn tại trong hệ thống hay không (chống account enumeration) — response phải giống nhau dù email tồn tại hay không.
- Token reset phải là random đủ mạnh, chỉ dùng được một lần, hết hạn trong khoảng thời gian ngắn (ví dụ 15-30 phút), và bị vô hiệu ngay khi user đặt mật khẩu mới thành công.
- Nếu user bấm nhiều lần vào nút "gửi lại link reset", chỉ token mới nhất còn hiệu lực — các token cũ phát trước đó phải bị vô hiệu ngay.
- Sau khi đổi mật khẩu thành công, phải tự động đăng xuất tất cả session đang hoạt động ở nơi khác (trừ thiết bị vừa thực hiện đổi), và gửi email thông báo "mật khẩu của bạn vừa được đổi" kèm cách báo cáo nếu không phải chính chủ.
- Rate-limit request reset password theo cả email và theo IP để chống spam gửi email hoặc brute-force dò token.
