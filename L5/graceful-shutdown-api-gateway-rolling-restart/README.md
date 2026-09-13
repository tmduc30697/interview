# API Gateway thực hiện rolling restart toàn cụm backend

**Hệ thống:** Một API Gateway đứng trước nhiều microservice của một nền tảng SaaS, cần rolling restart lần lượt các service trong giờ ít traffic mà không gây downtime cho khách hàng.

**Vai trò của flow:** Gateway phải phối hợp với connection draining của từng service để đảm bảo không có request nào bị route tới instance đang trong quá trình tắt.

**Yêu cầu cụ thể:**
- Gateway phải nhận được tín hiệu "instance X sắp shutdown" trước khi instance đó thực sự ngừng nhận connection (không dựa hoàn toàn vào health check polling định kỳ vốn có độ trễ phát hiện), để loại instance ra khỏi pool routing ngay lập tức.
- Với các kết nối HTTP keep-alive đang mở tới instance sắp tắt, gateway phải đảm bảo request tiếp theo trên kết nối đó (nếu có) được route sang instance khác, không tái sử dụng kết nối cũ tới instance đang chết.
- Trong lúc một service đang rolling restart (một số instance đã restart, một số chưa), phải xử lý được trường hợp version mới và version cũ trả về response có schema khác nhau nhẹ — gateway hoặc client không được crash vì field mới thiếu/thừa.
- Đặt cơ chế circuit breaker tạm thời tăng ngưỡng chịu lỗi (error budget) trong lúc rolling restart đang diễn ra, để tránh gateway hoảng loạn ngắt toàn bộ traffic chỉ vì vài request timeout do đúng lúc instance đang drain.
- Viết test toàn trình: rolling restart lần lượt 5 instance của một service trong lúc có traffic liên tục chạy nền, đo tỷ lệ lỗi/timeout trong toàn bộ quá trình phải bằng 0 hoặc dưới một ngưỡng SLA đã định nghĩa trước.
