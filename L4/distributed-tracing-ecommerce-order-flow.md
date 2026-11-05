# Truy vết một đơn hàng e-commerce qua giỏ hàng, tồn kho, thanh toán và xác nhận

**Hệ thống:** Một sàn e-commerce có luồng đặt hàng đi qua bốn service tách biệt: `cart-service` → `inventory-service` (kiểm tra và giữ tồn kho) → `payment-service` (gọi cổng thanh toán bên ngoài) → `order-confirmation-service`, mỗi service có tài nguyên và tốc độ scale khác nhau.

**Vai trò của flow:** Khi checkout bị chậm vào giờ cao điểm, tracing phải chỉ ra chính xác bước nào đang là nút thắt (tồn kho đang bị lock tranh chấp, hay cổng thanh toán bên ngoài đang chậm, hay chỉ đơn giản là hàng đợi nội bộ đang nghẽn) để đội vận hành không phải đoán mò giữa nhiều nguyên nhân khả dĩ.

**Yêu cầu cụ thể:**
- Trace của một đơn hàng phải giữ nguyên `trace_id` xuyên suốt bốn bước dù chúng chạy trên các service độc lập với vòng đời request khác nhau (một số đồng bộ, một số phải chờ callback bất đồng bộ từ cổng thanh toán), để dựng lại được toàn bộ hành trình của một đơn hàng cụ thể khi khách hàng báo lỗi.
- Bước kiểm tra tồn kho thường liên quan tới việc chờ lock (nhiều đơn hàng cùng tranh chấp một sản phẩm hot), span của bước này phải tách riêng thời gian chờ lock khỏi thời gian xử lý logic thật, để phân biệt được "chậm vì tranh chấp tồn kho" với "chậm vì code xử lý nặng".
- Bước gọi cổng thanh toán bên ngoài phải được đánh dấu rõ là external dependency với SLA riêng, để khi tổng hợp thống kê "checkout chậm ở bước nào" vào giờ cao điểm, đội vận hành không nhầm lẫn giữa độ trễ do chính hệ thống gây ra và độ trễ nằm ngoài tầm kiểm soát của cổng thanh toán đối tác.
- Thiết kế dashboard tổng hợp theo từng bước trên toàn bộ traffic (không chỉ theo từng đơn hàng riêng lẻ) để trả lời câu hỏi "giờ cao điểm hôm nay chậm hơn hôm qua, chậm chủ yếu ở bước nào" — cho phép so sánh phân phối độ trễ (p50/p95/p99) từng bước theo thời gian thực.
- Xử lý trường hợp một đơn hàng bị timeout ở bước thanh toán nhưng cổng thanh toán thực ra đã xử lý thành công phía họ và gửi callback trễ — trace phải ghi lại đủ để đối chiếu, tránh tình huống hệ thống báo "đơn hàng thất bại" trong khi thực tế tiền đã bị trừ, gây khiếu nại khó điều tra nếu thiếu dữ liệu trace đầy đủ.
