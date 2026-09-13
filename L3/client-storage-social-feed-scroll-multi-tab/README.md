# Khôi phục scroll feed và đồng bộ trạng thái đa tab cho mạng xã hội

**Hệ thống:** Mạng xã hội có feed dạng infinite scroll; khi user click vào 1 bài viết rồi bấm Back, cần khôi phục đúng vị trí scroll và danh sách bài đã load, đồng thời đồng bộ trạng thái đọc/like giữa nhiều tab cùng mở.

**Vai trò của flow:** Giữ trải nghiệm mượt (không phải load lại từ đầu feed khi quay lại) mà vẫn đảm bảo dữ liệu hiển thị không quá cũ so với hành động đã thực hiện ở tab khác.

**Yêu cầu cụ thể:**
- Lưu danh sách post đã load + vị trí scroll vào sessionStorage (theo tab, theo session) khi rời trang; khi quay lại bằng Back, khôi phục đúng danh sách và scroll position — nhưng phải giới hạn kích thước lưu (ví dụ chỉ giữ 50 post gần nhất) để tránh sessionStorage phình to khi user scroll rất sâu.
- Trình duyệt có bfcache (back/forward cache) khôi phục toàn bộ trang từ memory kể cả JS state mà không chạy lại code — phải xử lý đúng sự kiện `pageshow`/`persisted` để refresh những phần dữ liệu có thể đã đổi (số like, comment mới) thay vì hiển thị y nguyên trạng thái cũ đóng băng từ lúc rời trang.
- Khi user like/unlike một post ở tab A, cần đồng bộ trạng thái đó sang tab B đang mở cùng post đó (qua BroadcastChannel hoặc `storage` event) để tránh hiển thị lệch trạng thái like giữa 2 tab của cùng 1 user.
- Nếu post trong feed đã cache bị tác giả xóa hoặc chuyển private trong lúc user đang ở trang khác, khi quay lại từ cache phải phát hiện và loại bỏ post đó khỏi danh sách khôi phục thay vì hiển thị nội dung đã không còn quyền xem.
- Đo lường: tỷ lệ lần quay lại (Back) khôi phục thành công từ cache (không phải load lại từ API), so với số lần phải fallback load mới do cache miss/hết hạn/dữ liệu đã stale quá ngưỡng cho phép.
