# Checkout gọi payment gateway bên thứ ba

**Hệ thống:** Trang e-commerce gọi API cổng thanh toán bên ngoài (kiểu Stripe) để xử lý thanh toán lúc checkout.

**Vai trò của flow:** Circuit breaker + retry with backoff bảo vệ hệ thống khi payment gateway chậm/lỗi, tránh làm nghẽn toàn bộ luồng checkout và tránh gọi lại dồn dập làm gateway càng quá tải.

**Yêu cầu cụ thể:**
- Định nghĩa rõ ngưỡng mở circuit breaker (ví dụ tỉ lệ lỗi >50% trong 10 giây gần nhất, hoặc latency p95 vượt X ms) và hành vi khi breaker mở (fail-fast, không gọi gateway nữa trong một khoảng thời gian).
- Retry chỉ áp dụng cho lỗi có thể retry an toàn (timeout, 5xx) — tuyệt đối không retry tự động cho response không rõ kết quả liên quan tới việc charge tiền (tránh double-charge), phải có bước kiểm tra idempotency key trước khi retry.
- Backoff phải là exponential kèm jitter (ngẫu nhiên hóa) để tránh nhiều request cùng retry đồng loạt tạo ra "retry storm" làm gateway sập thêm.
- Circuit breaker phải có trạng thái half-open để thử lại một lượng nhỏ request sau khi hết thời gian mở, và chỉ đóng lại hoàn toàn khi các request thử nghiệm đó thành công.
- Đo lường: tỉ lệ request bị fail-fast do breaker mở, số lần breaker chuyển trạng thái trong ngày, và alert khi breaker mở quá lâu (báo hiệu sự cố kéo dài ở phía gateway).
