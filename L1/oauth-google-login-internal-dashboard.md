# "Đăng nhập bằng Google" cho một SaaS dashboard nội bộ

**Hệ thống:** Một dashboard quản lý task nội bộ (kiểu Trello mini) cho công ty, hiện chỉ có đăng nhập bằng email/password.

**Vai trò của flow:** Thêm nút "Sign in with Google" — app đóng vai trò OAuth Client, dùng authorization code flow để lấy identity token và định danh user, thay thế/song song với login truyền thống.

**Yêu cầu cụ thể:**
- Chỉ cho phép đăng nhập bằng các Google account thuộc domain công ty (chặn domain khác).
- Lần đầu login qua Google phải tự tạo user record và map với account cũ nếu email đã tồn tại (account linking).
- Phải dùng `state` param để chống CSRF, và validate `redirect_uri` khớp whitelist.
- Access token của Google chỉ dùng một lần để lấy profile — không lưu lại; hệ thống tự phát session token riêng (JWT/session cookie) sau khi xác thực xong.
- Xử lý được case user bấm "Deny" ở màn hình consent của Google.
