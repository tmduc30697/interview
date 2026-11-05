# Lưu token/session phía client cho ứng dụng ngân hàng số

**Hệ thống:** Ứng dụng ngân hàng số/fintech web lưu access token và dữ liệu phiên đăng nhập ở phía client, phải cân bằng giữa trải nghiệm (không bắt đăng nhập lại liên tục) và yêu cầu bảo mật nghiêm ngặt (không để token bị đánh cắp qua XSS, không rò rỉ khi dùng chung máy).

**Vai trò của flow:** Chọn đúng nơi lưu (memory/sessionStorage/httpOnly cookie) cho từng loại dữ liệu nhạy cảm, và xử lý đúng vòng đời khi đóng tab/đăng xuất/dùng chung máy.

**Yêu cầu cụ thể:**
- Access token ngắn hạn chỉ giữ trong memory (biến JS, không phải localStorage/sessionStorage) để giảm bề mặt tấn công XSS; refresh token dùng cookie httpOnly + Secure + SameSite do server set, JS không đọc trực tiếp được — nêu rõ vì sao localStorage bị loại cho trường hợp này (mọi script chạy trên trang, kể cả script bên thứ ba lỡ bị compromise, đều đọc được).
- Thông tin phi nhạy cảm (tab đang chọn, filter đang áp dụng trên dashboard) có thể lưu sessionStorage — nhưng phải tự xóa sạch mọi state khi logout, kể cả khi logout được trigger từ tab khác (đăng xuất ở 1 tab phải làm mất hiệu lực token ở toàn bộ tab đang mở, không chỉ tab bấm logout).
- Máy dùng chung (quầy giao dịch, máy tính công cộng): sessionStorage tự xóa khi đóng tab/trình duyệt là chưa đủ — cần thêm auto-logout theo thời gian không thao tác (idle timeout) và xóa rõ ràng toàn bộ cache/state khi timeout, không dựa hoàn toàn vào hành vi đóng tab của user.
- Khi vô tình mở 2 tab với 2 tài khoản khác nhau (nhân viên hỗ trợ dùng nhiều tài khoản test) — vì sessionStorage cô lập theo tab nên đây là hành vi đúng, nhưng phải test kỹ để đảm bảo không có state nào vô tình rò rỉ qua cơ chế khác (ví dụ BroadcastChannel dùng chung cho tính năng khác vô tình đồng bộ nhầm session giữa 2 tab khác tài khoản).
- Audit: liệt kê rõ mọi nơi dữ liệu nhạy cảm (số dư, số tài khoản, lịch sử giao dịch) có thể vô tình bị cache lại (browser back/forward cache - bfcache, hoặc cache của service worker nếu có) sau khi logout, và đảm bảo dùng `Cache-Control: no-store` hoặc unload handler để không xem lại được dữ liệu cũ bằng nút Back sau khi đã đăng xuất.
