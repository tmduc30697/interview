# Giữ giỏ hàng và form checkout dở dang qua localStorage

**Hệ thống:** SPA e-commerce lưu giỏ hàng và form checkout dở dang vào localStorage để user không mất dữ liệu khi reload/đóng nhầm tab, đồng thời phải đồng bộ đúng với giỏ hàng thật trên server khi đăng nhập.

**Vai trò của flow:** Đảm bảo trải nghiệm "không mất dữ liệu" khi mất mạng/đóng nhầm tab, nhưng không để dữ liệu cache phía client lấn át dữ liệu nguồn (server) gây sai lệch giá/tồn kho.

**Yêu cầu cụ thể:**
- Khi user thêm sản phẩm vào giỏ ở trạng thái chưa đăng nhập (guest), giỏ hàng lưu localStorage; khi đăng nhập, phải merge đúng với giỏ hàng đã có sẵn trên server của tài khoản đó — quyết định rõ thứ tự ưu tiên (giữ số lượng lớn hơn? cộng dồn? hỏi user khi conflict?) chứ không được ghi đè ngầm định gây mất dữ liệu.
- localStorage lưu snapshot giá tại thời điểm thêm vào giỏ; trước khi checkout phải revalidate lại giá/tồn kho thật với server, và xử lý rõ khi giá đổi hoặc hết hàng (hiển thị cảnh báo, không để user thanh toán nhầm giá cũ).
- Nhiều tab cùng mở giỏ hàng: khi 1 tab thêm/xóa item, cập nhật đồng bộ badge số lượng ở các tab khác qua `storage` event (hoặc BroadcastChannel), tránh tình trạng 1 tab ghi đè localStorage của tab khác làm mất thay đổi vừa thực hiện ở tab kia.
- Xử lý khi localStorage đầy (quota exceeded) hoặc bị user tắt qua chế độ duyệt riêng tư/tắt storage — có fallback (in-memory only, cảnh báo user) mà không crash trang.
- Dữ liệu form checkout dở dang (địa chỉ, note) có TTL tự hết hạn (ví dụ sau 24h) để tránh hiển thị lại thông tin đã lỗi thời (địa chỉ cũ, khuyến mãi hết hạn) như là dữ liệu hiện tại.
