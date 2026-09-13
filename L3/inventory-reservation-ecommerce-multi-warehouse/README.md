# Giữ tồn kho khi thêm vào giỏ hàng trên sàn e-commerce nhiều kho

**Hệ thống:** Một sàn e-commerce có tồn kho phân bổ ở nhiều kho khác nhau theo khu vực, khách hàng thêm sản phẩm vào giỏ và tồn kho được "soft reserve" trong 15 phút khi vào bước checkout.

**Vai trò của flow:** Flow reservation phải chọn đúng kho gần khách để giữ hàng, trừ tồn kho tạm thời (không phải trừ thật) khi vào checkout, và giải phóng đúng nếu khách rời trang mà không thanh toán.

**Yêu cầu cụ thể:**
- Khi 2 khách ở 2 khu vực khác nhau cùng checkout sản phẩm chỉ còn 1 đơn vị tồn kho ở kho gần cả hai, dùng update nguyên tử `UPDATE inventory SET reserved = reserved + 1 WHERE available - reserved > 0` (không phải đọc số dư trước rồi update sau) để chỉ 1 trong 2 giữ được hàng, người thua nhận thông báo ngay và được đề xuất kho khác (nếu có) hoặc thông báo hết hàng.
- Reservation phải có TTL và bảng lưu trạng thái riêng (không chỉ tăng/giảm số nguyên trong cột `reserved`), để có thể truy ngược "ai đang giữ, giữ bao lâu, khi nào hết hạn" phục vụ debug và hiển thị đúng cho khách.
- Mô tả cụ thể race giữa job giải phóng reservation hết hạn và request thanh toán đến gần đồng thời: transaction thanh toán phải kiểm tra reservation vẫn còn hiệu lực (chưa bị job expire xóa) ngay trước khi trừ tồn kho thật, nếu đã bị expire phải từ chối thanh toán rõ ràng và không charge tiền.
- Khi khách sửa số lượng trong giỏ hàng (tăng từ 1 lên 3) trong lúc đang giữ reservation, phải tính lại đúng phần chênh lệch cần giữ thêm một cách atomic, không phải hủy reservation cũ rồi tạo mới (dễ mất tồn kho đã giữ trong khoảng thời gian giữa 2 bước).
- Có cơ chế đồng bộ giữa tồn kho "hiển thị công khai trên trang sản phẩm" và tồn kho "khả dụng thực" (available - reserved), quy định độ trễ chấp nhận được và tránh hiển thị "còn hàng" khi thực chất đã được reserve hết bởi các giỏ hàng khác.
