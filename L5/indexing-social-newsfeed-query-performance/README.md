# Mạng xã hội tối ưu truy vấn newsfeed theo follow và thời gian

**Hệ thống:** Mạng xã hội hiển thị newsfeed gồm bài viết từ những người user đang follow, sắp xếp theo thời gian đăng và/hoặc mức độ liên quan.

**Vai trò của flow:** Chọn đúng chiến lược index (và ranh giới khi nào chuyển từ pull-query sang fan-out/push) để trả feed nhanh dù số lượng người follow và tổng số bài viết tăng lớn.

**Yêu cầu cụ thể:**
- Cách tiếp cận pull (JOIN bảng follows với posts, ORDER BY created_at) cần composite index (followee_id, created_at) trên bảng posts, nhưng với user follow rất nhiều tài khoản hoặc follow một celebrity có lượng bài viết khổng lồ, JOIN + index vẫn không đủ nhanh — cần xác định ngưỡng khi nào nên chuyển sang mô hình fan-out/push (ghi sẵn feed) thay vì tiếp tục tối ưu index cho pull-query.
- Bài viết bị xoá hoặc ẩn phải biến mất khỏi feed ngay lập tức, nhưng nếu chỉ dựa vào cột `is_deleted`/`is_hidden` không nằm trong index thì mỗi lần lọc vẫn phải quét thêm — cần đưa các cột trạng thái này vào composite/partial index để loại bỏ ngay ở tầng index thay vì lọc sau khi đã lấy dữ liệu.
- Khi feed cần xếp hạng theo độ liên quan (kết hợp engagement score, không chỉ theo thời gian đăng), điểm số này thay đổi liên tục theo lượt like/comment mới — B-tree index không tối ưu để sort theo một giá trị luôn biến động, cần cân nhắc pre-computed/materialized ranking được cập nhật định kỳ thay vì tính và sort real-time trên index thời gian.
- Ở các sự kiện viral, lượng bài viết mới insert tăng đột biến khiến index trên bảng posts (đặc biệt phần cuối theo created_at) bị ghi liên tục dẫn tới page split và phân mảnh — cần tính tới việc tuning fillfactor hoặc lịch bảo trì/rebuild index định kỳ để giữ hiệu năng ổn định.
- Danh sách follow/unfollow của user thay đổi liên tục, và bảng follows được truy vấn theo cả hai chiều (tìm những người tôi follow, và tìm những ai đang follow tôi) — cần quyết định thứ tự cột index (follower_id, followee_id) hay ngược lại dựa trên chiều truy vấn nào phổ biến hơn trong hệ thống, hoặc chấp nhận tạo cả hai index đánh đổi lấy overhead ghi.
