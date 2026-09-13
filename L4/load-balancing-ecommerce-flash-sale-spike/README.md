# E-commerce chịu tải đột biến trong flash sale

**Hệ thống:** Một sàn e-commerce có checkout service, trong sự kiện flash sale traffic có thể tăng 20 lần trong vài giây.

**Vai trò của flow:** Load balancer phải tự chuyển đổi hành vi giữa các thuật toán (round-robin lúc bình thường, least-connection khi có instance chạy job nặng) và bảo vệ backend khỏi bị sập toàn cụm.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế phát hiện một instance "chậm" (latency tăng cao) dù vẫn healthy theo health check thông thường, và giảm trọng số route tới instance đó (outlier detection), tránh trường hợp health check pass nhưng response time đã tệ.
- Khi toàn bộ cluster đạt ngưỡng tải tối đa, phải có cơ chế shed load có kiểm soát (trả lỗi 503 kèm `Retry-After` cho một phần request) thay vì để tất cả request timeout đồng loạt.
- Đảm bảo thuật toán LB không route request "mua hàng" (ghi dữ liệu, cần nhất quán) và request "xem sản phẩm" (đọc, có thể cache) theo cùng một cách — phân tách được pool instance hoặc trọng số theo loại traffic.
- Test race condition: nhiều request cùng cập nhật bộ đếm connection/tải của một instance đồng thời — đảm bảo số liệu dùng để ra quyết định route không bị lệch do cập nhật không atomic.
- Có cơ chế circuit breaker phối hợp với LB: khi một instance liên tục trả lỗi 5xx, tự động loại khỏi vòng quay trong một khoảng thời gian rồi thử lại dần (half-open), không chờ health check thủ công.
