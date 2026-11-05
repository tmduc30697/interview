# Canary deployment cho luồng checkout của sàn e-commerce trong giờ cao điểm

**Hệ thống:** Một sàn e-commerce có luồng checkout tách riêng khỏi các dịch vụ khác (dashboard quản trị, catalog, tìm kiếm), xử lý trực tiếp thanh toán và ảnh hưởng doanh thu theo từng phút.

**Vai trò của flow:** Canary ở đây phải giới hạn phạm vi đúng vào service checkout, tăng tỷ lệ traffic rất thận trọng, và có khả năng phát hiện + rollback trong vài chục giây vì mỗi phút lỗi thanh toán đều mất tiền thật, khác hẳn với canary một dashboard nội bộ ít rủi ro.

**Yêu cầu cụ thể:**
- Giai đoạn canary đầu tiên chỉ nên nhận một tỷ lệ traffic rất nhỏ (ví dụ dưới 1%) và ưu tiên chọn ngẫu nhiên theo request chứ không theo user cố định, để một lỗi ở phiên bản mới không lặp lại nhiều lần trên cùng một khách hàng gây trải nghiệm tệ liên tục.
- Định nghĩa metric rollback riêng cho checkout khác với metric kỹ thuật thông thường — không chỉ tỷ lệ lỗi 5xx mà cả tỷ lệ giao dịch thanh toán thất bại/timeout ở cổng thanh toán, vì phiên bản mới có thể trả về 200 OK nhưng vẫn gọi sai tham số khiến cổng thanh toán từ chối giao dịch.
- Xử lý trường hợp một đơn hàng đã bắt đầu ở phiên bản canary (đã giữ tồn kho, đã tạo intent thanh toán) nhưng canary bị rollback giữa chừng do phát hiện lỗi — đơn hàng đó phải được hoàn tất hoặc hoàn tác một cách nhất quán, không để đơn ở trạng thái treo dở dang khi traffic chuyển hết về phiên bản cũ.
- Đảm bảo cơ chế giám sát tỷ lệ lỗi thanh toán phải tính riêng theo phiên bản (canary vs stable) theo thời gian gần thực (vài giây tới một phút), vì đợi tổng hợp báo cáo theo giờ là quá chậm để tránh thiệt hại doanh thu khi canary có lỗi.
- Thiết kế cơ chế rollback tự động không cần chờ người trực ca xác nhận thủ công đối với các ngưỡng lỗi nghiêm trọng đã định nghĩa trước (ví dụ tỷ lệ thanh toán thất bại tăng gấp đôi so với baseline trong 2 phút), vì tốc độ phản ứng ở đây quan trọng hơn quy trình phê duyệt nhiều bước.
