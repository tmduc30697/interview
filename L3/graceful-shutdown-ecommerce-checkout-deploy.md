# Checkout service của sàn e-commerce trong lúc deploy

**Hệ thống:** Một service xử lý checkout (tạo order, trừ tồn kho, gọi cổng thanh toán) cho sàn e-commerce, được deploy nhiều lần mỗi ngày.

**Vai trò của flow:** Khi rolling deploy loại bỏ instance cũ, instance đó phải hoàn tất các giao dịch checkout đang xử lý trước khi tắt, không được cắt ngang giữa lúc đang trừ tồn kho hoặc gọi cổng thanh toán.

**Yêu cầu cụ thể:**
- Khi nhận SIGTERM, instance phải ngay lập tức báo readiness = false (để load balancer ngừng gửi request mới) nhưng vẫn tiếp tục xử lý các request đang chạy cho tới khi xong hoặc hết grace period.
- Định nghĩa rõ grace period tối đa (ví dụ 30 giây) — nếu một giao dịch checkout chưa xong khi hết grace period, phải có chiến lược rõ ràng (log lại state, không để order ở trạng thái "nửa vời" — ví dụ đã trừ tồn kho nhưng chưa tạo order) thay vì kill cứng.
- Xử lý trường hợp instance đang giữ một distributed lock (ví dụ lock tồn kho sản phẩm) khi bị shutdown — phải release lock hoặc để lock tự hết TTL, không được để lock treo vĩnh viễn làm nghẽn các checkout khác.
- Đảm bảo trong lúc drain, instance không nhận thêm request mới (health check đã báo not-ready) nhưng vẫn phải trả lời được health check probe để không bị hạ tầng kill ngay lập tức trước khi drain xong.
- Viết test mô phỏng gửi 100 request checkout đồng thời rồi trigger shutdown giữa chừng, verify không có request nào bị mất kết nối nửa đường (mỗi request phải nhận được response, kể cả response lỗi rõ ràng, không bao giờ là timeout im lặng).
