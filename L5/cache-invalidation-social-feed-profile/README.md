# Cache feed/trang cá nhân cho mạng xã hội

**Hệ thống:** Mạng xã hội cache feed trang chủ và trang cá nhân (profile) của user để giảm tải khi hiển thị bài viết, tránh phải chạy lại aggregation nặng mỗi lần load.

**Vai trò của flow:** Invalidate đúng cache khi user đăng bài mới, xóa bài, sửa bài, đổi thông tin profile — đặc biệt phải fan-out invalidate/refresh cho cache feed của tất cả follower đang xem nội dung của user đó.

**Yêu cầu cụ thể:**
- Khi user đăng bài mới, cache feed của chính họ (profile page) phải invalidate ngay, nhưng cache feed của hàng chục nghìn follower thì không thể invalidate đồng bộ tức thời — cần chiến lược phân biệt giữa fan-out-on-write (đẩy trước) cho user ít follower và fan-out-on-read (tính khi đọc) cho celebrity account có follower cực lớn, tránh nghẽn khi 1 bài đăng làm invalidate hàng triệu cache feed cùng lúc.
- Khi user xóa 1 bài viết, phải đảm bảo bài đó biến mất khỏi cache feed của follower trong thời gian hợp lý dù dùng fan-out-on-read hay pull — không được để bài đã xóa vẫn hiển thị dai dẳng do cache feed của follower ít khi active nên hiếm khi được refresh.
- Race condition: user vừa đăng bài mới vừa sửa bài cũ gần như đồng thời — invalidation event có thể tới cache theo thứ tự khác thứ tự thực tế (do khác hàng đợi/khác node xử lý), cần cơ chế đảm bảo thứ tự (version/timestamp) để cache feed không áp dụng nhầm invalidate cũ đè lên update mới.
- Khi user đổi avatar/tên hiển thị, các bài post cũ của họ đang nằm rải rác trong cache feed của follower khác vẫn hiển thị thông tin cũ (do cache feed thường lưu kèm snapshot info tác giả) — quyết định rõ có chấp nhận độ trễ hiển thị (denormalized data lệch tạm thời) hay bắt buộc phải invalidate toàn bộ cache liên quan, đánh đổi chi phí fan-out rất lớn.
- Đo lường: độ trễ trung bình và p99 từ lúc user đăng/xóa bài tới lúc feed của follower phản ánh đúng, và tỉ lệ follower vẫn thấy nội dung cũ quá X phút sau invalidation event, dùng để đánh giá hiệu quả chiến lược fan-out đang chọn.
