# Đặt phòng khách sạn với cập nhật giá và tồn phòng cùng lúc

**Hệ thống:** Một nền tảng đặt phòng khách sạn, admin khách sạn có thể cập nhật giá/số phòng trống trong khi khách đang đặt phòng.

**Vai trò của flow:** Transaction đặt phòng (trừ tồn phòng) và transaction admin cập nhật giá/tồn phòng cùng chạm vào bảng `room_inventory`, cần chọn isolation level và thứ tự lock hợp lý để không deadlock và không đọc dữ liệu giá sai (stale).

**Yêu cầu cụ thể:**
- Định nghĩa rõ transaction đặt phòng phải `SELECT ... FOR UPDATE` trên đúng các row (room_type, date) cần trừ tồn, không lock nguyên bảng.
- Nếu một request đặt phòng theo thứ tự ngày [10/9, 11/9, 12/9] và request khác đặt theo thứ tự [12/9, 11/9, 10/9] (đặt nhiều đêm cùng lúc), phải chuẩn hóa thứ tự lock theo khóa chính tăng dần trước khi lock để loại bỏ deadlock giữa 2 booking đa-ngày chồng lấn.
- So sánh và chọn isolation level: dùng `READ COMMITTED` + lock tường minh cho phần trừ tồn, tránh dùng `SERIALIZABLE` toàn bộ vì sẽ làm tăng abort rate không cần thiết ở phần đọc giá hiển thị cho khách đang xem trang.
- Khi admin sửa giá đúng lúc khách đang giữ lock trừ tồn cho phòng đó, request admin phải chờ hoặc bị timeout rõ ràng (không treo vô hạn) — quy định timeout cụ thể (ví dụ 3 giây) và hành vi khi timeout (báo lỗi "thử lại sau").
- Viết test dựng 2 transaction giả lập chồng lấn lock theo thứ tự ngược nhau, xác nhận hệ thống tự phát hiện/rollback một bên và request còn lại thành công.
