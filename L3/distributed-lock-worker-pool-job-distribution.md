# Phân việc cho worker pool xử lý hàng đợi công việc lớn

**Hệ thống:** Nhiều worker instance chạy song song để xử lý hàng loạt job từ một hàng đợi công việc lớn (ví dụ xử lý file, gửi batch email), cần chia việc sao cho không trùng lặp và không bỏ sót.

**Vai trò của flow:** Distributed lock/leader election theo từng job (hoặc theo phân vùng job) đảm bảo mỗi job chỉ được đúng một worker nhận và xử lý tại một thời điểm, đồng thời không bỏ sót job khi worker chết giữa chừng.

**Yêu cầu cụ thể:**
- Worker phải "claim" một job bằng lock/lease có TTL trước khi bắt đầu xử lý; nếu worker chết giữa chừng (crash, bị kill do autoscale down) mà không release lock, TTL hết hạn phải tự đưa job đó về trạng thái "khả dụng" để worker khác nhận lại — không để job kẹt vĩnh viễn ở trạng thái "đang xử lý" bởi worker đã chết.
- Hai worker cùng lúc thấy job "chưa ai nhận" và cùng cố gắng claim (race condition khi liệt kê job và acquire lock không atomic với nhau) phải được giải quyết sao cho chỉ đúng một worker thắng, worker thua phải nhận biết ngay để không tiếp tục xử lý song song, tránh xử lý trùng và ghi kết quả hai lần.
- Nếu job cần thời gian xử lý lâu hơn TTL ước tính ban đầu (ví dụ file lớn bất thường), worker đang xử lý phải renew lease định kỳ; nếu renew thất bại (mất kết nối coordinator) trong khi job vẫn đang chạy thật, cần cơ chế idempotent hoặc kiểm tra trạng thái trước khi apply kết quả để chịu được việc worker khác nhận lại job đó khi lock hết hạn.
- Khi một worker bị autoscale terminate có báo trước (graceful shutdown signal), phải chủ động release lock/trả lại job đang giữ ngay lập tức thay vì chờ TTL hết hạn, để giảm độ trễ job bị "treo" chờ timeout tự nhiên.
- Cần cơ chế định kỳ quét các job ở trạng thái "đang xử lý" quá lâu so với TTL kỳ vọng (dead job detection) để phát hiện sớm rò rỉ lock hoặc lỗi hệ thống, tránh dựa hoàn toàn vào TTL tự nhiên mà không có giám sát chủ động.
