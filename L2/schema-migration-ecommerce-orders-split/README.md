# Tách bảng `orders` thành `orders` + `order_items` không downtime (e-commerce)

**Hệ thống:** Một sàn e-commerce có bảng `orders` cũ lưu items dưới dạng JSON trong 1 cột, cần tách sang bảng quan hệ `order_items` để hỗ trợ báo cáo/join hiệu quả hơn.

**Vai trò của flow:** Trong lúc traffic đặt hàng vẫn chạy 24/7, flow phải đảm bảo mọi order mới tạo được ghi cả vào cột JSON cũ và bảng `order_items` mới, đồng thời có job backfill order cũ, mà không làm gián đoạn checkout.

**Yêu cầu cụ thể:**
- Yêu cầu transaction tạo order dual-write phải wrap trong 1 transaction DB duy nhất (ghi `orders` + insert `order_items`) để không có trường hợp order được tạo nhưng thiếu order_items do crash giữa chừng.
- Mô tả cụ thể tình huống 2 request checkout đồng thời cho cùng 1 cart (double-submit do user bấm 2 lần) trong giai đoạn dual-write: yêu cầu idempotency key theo `cart_id` + `checkout_attempt_id` để chỉ 1 trong 2 request tạo order thành công, request kia phải nhận lại kết quả của request đầu (không tạo order trùng ở cả bảng cũ và mới).
- Job backfill order lịch sử phải chạy bằng cursor theo `order_id` tăng dần theo batch nhỏ (ví dụ 500 order/lần), có checkpoint để resume nếu job bị crash giữa chừng, không đọc lại từ đầu.
- Định nghĩa cơ chế kiểm tra tính nhất quán (reconciliation job) so sánh số lượng/tổng tiền giữa cột JSON cũ và bảng `order_items` mới cho một mẫu order ngẫu nhiên mỗi ngày, cảnh báo nếu phát hiện lệch.
- Quy định rõ điều kiện để an toàn drop cột JSON cũ: phải chạy dual-write ổn định (không lệch dữ liệu) trong một khoảng thời gian tối thiểu, và đã chuyển 100% luồng đọc sang bảng mới.
