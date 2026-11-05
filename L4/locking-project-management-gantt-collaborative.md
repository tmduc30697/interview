# Chỉnh sửa kế hoạch dự án (Gantt chart) trong tool quản lý dự án cộng tác

**Hệ thống:** Một tool quản lý dự án (kiểu Asana/Jira) cho phép nhiều thành viên cùng chỉnh sửa lịch trình task, gán người phụ trách, đổi ngày deadline trên cùng 1 dự án.

**Vai trò của flow:** Vì các thành viên thường sửa các task khác nhau (ít đụng cùng 1 task) nhưng đôi khi 2 người cùng sửa 1 task, flow nên dùng optimistic locking ở cấp độ từng task (không lock cả dự án) để tối đa hóa khả năng làm việc song song.

**Yêu cầu cụ thể:**
- Lock optimistic phải áp dụng ở cấp độ từng task (mỗi task có version riêng), không phải ở cấp độ toàn dự án — để thành viên A sửa task 1 và thành viên B sửa task 2 không bị chặn/xung đột lẫn nhau dù đang cùng mở dự án.
- Mô tả cụ thể: thành viên A đổi deadline task X từ ngày 10 sang ngày 15 (đọc task ở version 3), đồng thời thành viên B gán thêm người phụ trách cho cùng task X (cũng đọc ở version 3) — cả 2 đều sửa các trường khác nhau nhưng cùng version, ai lưu trước sẽ thắng và tăng version, người lưu sau bị từ chối theo optimistic lock thông thường; bàn về việc có nên áp dụng field-level merge (merge các trường không trùng nhau) thay vì từ chối toàn bộ, và ưu/nhược điểm của việc đó.
- Khi task đang được kéo-thả để đổi thứ tự trên Gantt chart (thao tác kéo thả có thể gửi nhiều update nhỏ liên tiếp trong thời gian ngắn), cần quy định rõ có nên coi mỗi lần kéo-thả là 1 version riêng (dễ tạo conflict giả nếu 2 người cùng kéo-thả các task khác nhau ảnh hưởng tới thứ tự chung) hay dùng cơ chế khác (ví dụ optimistic lock ở cấp "toàn bộ thứ tự danh sách task" thay vì từng task riêng khi liên quan tới sắp xếp).
- Nếu task bị xóa bởi 1 thành viên đúng lúc thành viên khác đang sửa task đó (gửi update dựa trên version cũ trước khi bị xóa), request update phải nhận lỗi rõ ràng "task đã bị xóa", không tạo lại task đã xóa hoặc gây lỗi không rõ nguyên nhân.
- Với các trường có tần suất tranh chấp cao hơn (ví dụ % hoàn thành task được nhiều người cùng cập nhật liên tục), đánh giá xem optimistic lock có phù hợp hay nên chuyển sang cơ chế khác (ví dụ increment atomic riêng cho trường đó) để giảm tỷ lệ conflict phải retry liên tục gây trải nghiệm khó chịu.
