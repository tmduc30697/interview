# Chỉnh sửa bài viết CMS bởi nhiều biên tập viên

**Hệ thống:** Một CMS quản lý nội dung cho tòa soạn báo online, nhiều biên tập viên có thể cùng mở và sửa 1 bài viết (ít xảy ra nhưng không hiếm, ví dụ 2 biên tập viên cùng ca trực).

**Vai trò của flow:** Vì xung đột sửa đồng thời hiếm xảy ra nhưng cần trải nghiệm mượt (không muốn khóa bài viết chặn người khác chỉ vì 1 người đang mở xem), flow lưu bài viết nên dùng optimistic locking dựa trên version, phát hiện xung đột tại thời điểm lưu thay vì chặn ngay từ khi mở bài.

**Yêu cầu cụ thể:**
- Mỗi bài viết có cột `version` tăng dần mỗi lần lưu; khi biên tập viên A mở bài ở version 5, sửa nội dung rồi lưu, request lưu phải gửi kèm version 5 đã đọc và server chỉ chấp nhận nếu version hiện tại trong DB vẫn là 5 (update kèm điều kiện `WHERE version = 5`), nếu không khớp phải trả lỗi conflict rõ ràng, không âm thầm ghi đè.
- Mô tả cụ thể: biên tập viên A và B cùng mở bài ở version 5, B lưu trước (bài chuyển sang version 6), A lưu sau vẫn gửi version 5 — request của A phải bị từ chối với thông báo "bài viết đã được người khác cập nhật", kèm hiển thị nội dung mới nhất (version 6) để A quyết định merge tay hoặc ghi đè có ý thức, không tự động merge ngầm.
- Cho phép A chọn "lưu đè" sau khi đã xem nội dung mới của B (ghi đè có ý thức, không phải ghi đè mù), hành động này phải tạo version mới dựa trên version hiện tại thực (6 → 7), không dùng lại version cũ (5) đã lỗi thời.
- Tự động lưu draft định kỳ (auto-save) trong lúc biên tập viên đang soạn không được tính là 1 "lưu chính thức" làm tăng version theo cách gây xung đột giả với người khác — quy định rõ auto-save lưu vào 1 bản riêng (draft riêng theo user) tách khỏi bản chính thức, chỉ khi publish mới áp dụng optimistic lock lên bản chính thức.
- Ghi log lịch sử các version bài viết (ai sửa, khi nào, nội dung gì thay đổi) để biên tập viên có thể xem lại/khôi phục version cũ nếu bị lưu đè nhầm.
