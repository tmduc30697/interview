# SaaS help center tìm kiếm ngữ nghĩa tài liệu trợ giúp

**Hệ thống:** Một SaaS có help center/tài liệu kỹ thuật cho khách hàng tự tra cứu, nội dung được đội support/docs chỉnh sửa thường xuyên.

**Vai trò của flow:** Lập chỉ mục nội dung bài viết trợ giúp, phục vụ tìm kiếm xếp hạng theo mức độ liên quan ngữ nghĩa (không chỉ khớp từ khóa) và đảm bảo cập nhật ngay khi nội dung được sửa để tránh hiển thị thông tin lỗi thời.

**Yêu cầu cụ thể:**
- Kết quả tìm kiếm phải xếp hạng được các bài viết diễn đạt khác từ ngữ nhưng cùng ý nghĩa với câu hỏi người dùng gõ (ví dụ hỏi "không đăng nhập được" phải tìm ra bài viết tiêu đề "khắc phục lỗi xác thực tài khoản"), đòi hỏi cơ chế đánh giá liên quan vượt ra ngoài khớp từ khóa thuần túy, đồng thời không được làm chậm đáng kể thời gian trả kết quả so với tìm kiếm từ khóa thông thường.
- Khi một bài viết được đội docs chỉnh sửa để sửa thông tin sai/lỗi thời, index phải phản ánh nội dung mới gần như ngay lập tức — độ trễ cập nhật kéo dài có thể khiến khách hàng tra cứu ra hướng dẫn đã biết là sai, gây thêm ticket support thay vì giảm.
- Bài viết ở trạng thái nháp/chưa xuất bản hoặc đã bị gỡ do lỗi nghiêm trọng phải không xuất hiện trong kết quả tìm kiếm, kể cả trong khoảng thời gian ngắn giữa lúc tác giả bấm lưu nháp và lúc job index xử lý xong, tránh rò rỉ nội dung chưa hoàn chỉnh/sai ra khách hàng.
- Khi nhiều phiên bản của cùng một bài viết tồn tại cho các gói/phiên bản sản phẩm khác nhau, kết quả tìm kiếm phải trả đúng phiên bản phù hợp với ngữ cảnh khách hàng đang xem, tránh gợi ý bài hướng dẫn cho tính năng khách hàng không có quyền sử dụng.
- Việc đội docs sửa hàng loạt bài viết cùng lúc (ví dụ đổi tên tính năng, rebrand) phải được xử lý qua pipeline index mà không tạo ra khoảng thời gian index ở trạng thái nửa cũ nửa mới hiển thị lẫn lộn cho các khách hàng khác nhau tìm kiếm cùng lúc.
