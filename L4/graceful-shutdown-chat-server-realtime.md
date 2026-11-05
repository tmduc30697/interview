# Chat server real-time cho ứng dụng nhắn tin

**Hệ thống:** Một backend chat real-time giữ kết nối WebSocket cho hàng chục nghìn user, mỗi kết nối gắn với một session hội thoại.

**Vai trò của flow:** Đảm bảo khi một chat server instance cần tắt (maintenance/deploy), user không bị mất tin nhắn đang gửi và được chuyển êm sang instance khác.

**Yêu cầu cụ thể:**
- Tin nhắn đang trong hàng đợi gửi (chưa được ACK bởi client) tại thời điểm shutdown phải được đảm bảo không mất — chuyển giao lại cho hàng đợi bền (message queue/DB) để instance khác tiếp tục gửi, không được chỉ giữ trong memory rồi mất khi tắt.
- Khi ngắt kết nối WebSocket để drain, phải gửi kèm mã lý do (close code) riêng cho "server draining" khác với "lỗi bất thường", để client phân biệt và tự reconnect ngay, không hiển thị lỗi gây hoang mang cho người dùng.
- Đảm bảo tính idempotent: nếu một tin nhắn đã được gửi thành công tới client trước khi server bị ngắt giữa lúc gửi ACK, khi reconnect vào instance mới, tin nhắn không được gửi lại trùng lặp.
- Xử lý trường hợp user đang gõ (typing indicator) hoặc đang trong cuộc gọi voice/video khi server drain — trạng thái này phải được chuyển giao hoặc kết thúc rõ ràng, không để lại trạng thái "đang gõ" treo vĩnh viễn ở phía người nhận.
- Thiết kế cơ chế rolling restart cho toàn cluster chat server sao cho tại mọi thời điểm, tổng số kết nối bị buộc reconnect đồng thời không vượt quá X% tổng số user online, tránh gây spike tải lên hệ thống presence/auth khi hàng loạt client reconnect cùng lúc.
