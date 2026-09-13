# Nền tảng chia sẻ video kiểu YouTube

**Hệ thống:** Một nền tảng cho phép user upload video và người khác xem lại (video-on-demand), quy mô hàng triệu video.

**Vai trò của flow:** Nhận file video gốc từ user, transcode ra nhiều độ phân giải/bitrate và đóng gói HLS/DASH để phát trên mọi thiết bị.

**Yêu cầu cụ thể:**
- Upload phải hỗ trợ file lớn (nhiều GB) bằng chunked/resumable upload, tiếp tục được nếu mất mạng giữa chừng.
- Sau khi upload xong, đưa vào hàng đợi transcode ra tối thiểu 3 độ phân giải (ví dụ 1080p/720p/480p) chạy song song, không block nhau nếu một rendition lỗi.
- Sinh thumbnail tự động (nhiều frame để user chọn) và file manifest HLS/DASH tương ứng.
- Nếu job transcode fail giữa chừng (worker crash, hết dung lượng tạm), phải retry lại đúng phần việc còn thiếu, không transcode lại từ đầu toàn bộ video.
- Video chỉ hiển thị "public" sau khi tối thiểu 1 rendition sẵn sàng; các rendition còn lại có thể hoàn thành dần (progressive availability).
- Có cơ chế dọn file gốc/file tạm sau X ngày để tối ưu chi phí storage, nhưng vẫn giữ được khả năng re-transcode nếu cần đổi codec sau này.
