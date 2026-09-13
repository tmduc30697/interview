# Background worker xử lý job dài đang consume message từ queue

**Hệ thống:** Một cụm worker consume message từ hàng đợi để xử lý các job chạy lâu (export file dữ liệu lớn, gửi email hàng loạt, xử lý batch report), mỗi job có thể mất từ vài chục giây tới vài phút để hoàn tất.

**Vai trò của flow:** Khi worker bị shutdown (deploy, autoscale scale-in), phải đảm bảo message đang xử lý dở không bị mất và không bị ack sai thời điểm — ack quá sớm làm mất dữ liệu nếu worker chết giữa chừng, ack quá muộn hoặc không ack làm message bị xử lý lặp lại không cần thiết.

**Yêu cầu cụ thể:**
- Khi nhận tín hiệu shutdown, worker phải ngay lập tức ngừng nhận message mới từ queue (ngừng poll/prefetch) nhưng vẫn tiếp tục xử lý cho tới khi hoàn tất các message đã nhận trước đó, tránh tình huống nhận thêm việc mới ngay trước lúc tắt rồi phải bỏ dở.
- Message chỉ được ack (xác nhận đã xử lý xong, xóa khỏi queue) sau khi job đã hoàn tất toàn bộ và kết quả đã được ghi bền vững — nếu worker bị kill cứng giữa lúc xử lý (hết grace period), message chưa ack phải tự động được đưa trở lại queue để worker khác xử lý lại, không được để mất.
- Với các job có thể để lại tác động một phần khi bị ngắt giữa chừng (ví dụ export file đã ghi được nửa file, gửi email hàng loạt đã gửi được một số người trong danh sách), phải thiết kế idempotent hoặc có checkpoint để lần xử lý lại sau không lặp lại phần đã hoàn tất (không gửi trùng email cho người đã nhận, không tạo file export bị lỗi nửa vời).
- Định nghĩa grace period đủ dài để job dài nhất trong hệ thống có cơ hội hoàn tất bình thường (không cắt ngang job đang chạy gần xong), nhưng vẫn có ngưỡng tối đa hợp lý để không làm chậm quá trình deploy/scale toàn cụm — cân nhắc cho phép job vượt quá một độ dài nhất định tự chủ động lưu checkpoint giữa chừng thay vì cố chờ nó chạy xong trong grace period.
- Xử lý trường hợp toàn bộ worker trong cụm đều nhận tín hiệu shutdown gần như đồng thời (ví dụ rolling deploy toàn cụm) — phải đảm bảo còn ít nhất một số worker tiếp tục nhận message mới trong lúc số còn lại đang drain, tránh để hàng đợi bị dồn ứ không ai xử lý trong lúc chuyển giao.
