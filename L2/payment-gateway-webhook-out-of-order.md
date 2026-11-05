# Thanh toán qua cổng bên thứ ba với webhook callback không đảm bảo thứ tự

**Hệ thống:** Một sàn e-commerce tích hợp cổng thanh toán bên thứ ba (kiểu Stripe/VNPay), nhận callback (webhook) báo kết quả giao dịch sau khi khách thanh toán.

**Vai trò của flow:** Flow xử lý callback phải cập nhật đúng trạng thái đơn hàng dựa trên webhook nhận được, kể cả khi webhook bị gửi trùng nhiều lần hoặc đến không đúng thứ tự thời gian thực tế xảy ra.

**Yêu cầu cụ thể:**
- Webhook phải được xử lý idempotent theo `transaction_id` của cổng thanh toán: nếu nhận cùng 1 webhook 3 lần (do cổng thanh toán retry vì response chậm), chỉ áp dụng thay đổi trạng thái đúng 1 lần, 2 lần sau chỉ trả 200 OK mà không làm gì thêm.
- Mô tả cụ thể tình huống thứ tự đảo lộn: webhook "payment_success" đến sau webhook "payment_refunded" của cùng giao dịch (do độ trễ mạng khác nhau) — yêu cầu dùng timestamp hoặc sequence number từ phía cổng thanh toán để xác định thứ tự thật, không dùng thời điểm hệ thống nhận được webhook để quyết định trạng thái cuối cùng.
- Khi webhook báo thành công đến gần như đồng thời với việc khách bấm "Hủy đơn" ở app, quy định rõ ai thắng: nếu tiền đã thực sự được trừ ở cổng thanh toán (webhook success là nguồn sự thật), đơn hàng không được hủy mà phải chuyển sang luồng hoàn tiền, không đơn giản đảo trạng thái theo request đến sau.
- Nếu server nội bộ bị down đúng lúc cổng thanh toán gửi webhook, phải đảm bảo cổng thanh toán sẽ retry theo cơ chế của họ (hoặc tự chủ động polling trạng thái giao dịch định kỳ như phương án dự phòng) — không để đơn hàng bị "treo" vĩnh viễn ở trạng thái "đang xử lý" chỉ vì lỡ mất đúng 1 webhook.
- Có job đối soát hàng ngày so sánh danh sách giao dịch thành công ghi nhận nội bộ với báo cáo từ cổng thanh toán, phát hiện và cảnh báo các giao dịch lệch (có ở 1 bên nhưng không có ở bên kia).
