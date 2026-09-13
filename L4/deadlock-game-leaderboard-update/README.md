# Cập nhật điểm số và thứ hạng leaderboard trong game/app học tập

**Hệ thống:** Một app học tập gamification, nơi hoàn thành bài học cập nhật điểm user và một job nền tính lại thứ hạng leaderboard theo nhóm/lớp mỗi vài phút.

**Vai trò của flow:** Transaction cộng điểm khi user hoàn thành bài học và job tính lại rank (đọc/update nhiều user trong 1 nhóm) cạnh tranh lock trên bảng `user_score`, cần tránh deadlock giữa update đơn lẻ tần suất cao và job batch định kỳ.

**Yêu cầu cụ thể:**
- Yêu cầu job tính rank phải lock theo thứ tự user_id tăng dần trong từng nhóm, và xử lý theo từng nhóm nhỏ (không lock toàn bảng `user_score` một lần) để giảm collision với các transaction cộng điểm đang chạy real-time.
- Chỉ ra rủi ro cụ thể: nếu 2 user trong cùng nhóm cùng hoàn thành bài học đúng lúc job rank đang chạy, cả 2 transaction cộng điểm và transaction job đều có thể chờ nhau — yêu cầu retry tự động cho transaction cộng điểm (ưu tiên UX real-time của user) khi gặp deadlock, tối đa 2 lần trong 500ms.
- Chọn isolation level `READ COMMITTED` cho transaction cộng điểm để có UX nhanh, nhưng job tính rank cần đảm bảo tính nhất quán tương đối (không bắt buộc real-time chính xác tuyệt đối) — giải thích trade-off này bằng ví dụ cụ thể.
- Thiết kế cơ chế "lock timeout" ngắn (ví dụ 1 giây) cho transaction cộng điểm, để nếu job rank đang giữ lock lâu, user vẫn nhận được phản hồi (có thể là "điểm sẽ cập nhật sau ít giây") thay vì trải nghiệm bị treo.
- Đo và log tỷ lệ deadlock/lock-timeout theo giờ cao điểm (khi nhiều user học cùng lúc) để xác định xem có cần điều chỉnh tần suất chạy job rank hay không.
