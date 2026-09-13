# API gateway public cho SaaS đa khách hàng (per-API-key rate limit)

**Hệ thống:** SaaS cung cấp public API cho khách hàng dùng qua API key, cần giới hạn số request theo từng gói dịch vụ (free/pro/enterprise).

**Vai trò của flow:** Rate limiting đảm bảo công bằng tài nguyên giữa các khách hàng, tránh một khách hàng dùng quá nhiều làm ảnh hưởng khách khác, đồng thời đúng với gói dịch vụ họ trả tiền.

**Yêu cầu cụ thể:**
- Mỗi API key có giới hạn request/giây và request/tháng riêng theo gói, và response khi vượt limit phải trả đúng chuẩn (HTTP 429 kèm header Retry-After, X-RateLimit-Remaining).
- Dùng thuật toán rate limit cụ thể (ví dụ token bucket hoặc sliding window log) để cho phép burst ngắn hợp lý mà không cho phép lạm dụng liên tục vượt giới hạn trung bình.
- Rate limit counter phải chính xác trong môi trường nhiều instance API gateway chạy song song (không được để mỗi instance tự đếm riêng dẫn tới tổng vượt xa giới hạn thực).
- Khi Redis/coordinator lưu counter bị chậm/down tạm thời, phải có chiến lược rõ ràng (fail-open cho phép qua tạm hay fail-closed chặn tạm) và giải thích lý do chọn theo bối cảnh SaaS đó.
- Có dashboard cho khách hàng tự xem usage hiện tại so với limit gói của họ, để họ tự chủ động nâng cấp gói trước khi bị chặn.
