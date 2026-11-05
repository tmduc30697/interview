# Cache danh sách lớn offline-first bằng IndexedDB cho SaaS dashboard

**Hệ thống:** SaaS B2B dashboard hiển thị danh sách lớn (hàng chục nghìn bản ghi: task, ticket, contact) cần hoạt động mượt và vẫn xem được dữ liệu gần nhất khi mất mạng tạm thời, dùng IndexedDB làm cache tầng client song song với gọi API.

**Vai trò của flow:** Giữ app "cảm giác nhanh" (đọc từ IndexedDB trước, render ngay) đồng thời đồng bộ đúng khi có mạng trở lại và khi mạng không ổn định.

**Yêu cầu cụ thể:**
- Chiến lược cache: đọc từ IndexedDB hiển thị ngay (stale-while-revalidate), gọi API ngầm để lấy bản mới nhất và cập nhật lại UI + IndexedDB — không được để user chỉnh sửa dựa trên dữ liệu stale mà tưởng là mới nhất, phải có chỉ báo rõ ("đang cập nhật..." / thời điểm sync gần nhất).
- Đổi schema lưu trong IndexedDB giữa các version app (thêm field, đổi cấu trúc object store) — phải có cơ chế migration (`onupgradeneeded`) không làm mất dữ liệu cache cũ hoặc crash app cho user đang chạy version cũ chưa update.
- Ghi nhận thao tác user khi offline (sửa/xóa record) vào một "outbox" trong IndexedDB, replay đúng thứ tự lên server khi có mạng lại; xử lý conflict khi bản ghi đó đã bị người khác sửa trên server trong lúc offline.
- Giới hạn dung lượng IndexedDB (quota per-origin của trình duyệt) — có chiến lược eviction (LRU theo bảng/dự án ít dùng nhất) khi gần chạm quota, tránh lỗi ghi thất bại giữa chừng làm hỏng transaction đang dở.
- Đo lường: tỷ lệ cache hit của IndexedDB so với phải gọi API, và độ trễ giữa "hiển thị từ cache" tới "đồng bộ xong dữ liệu mới nhất", để biết cache có thực sự giúp ích hay đang gây hiển thị sai kéo dài.
