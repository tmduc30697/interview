# Truy vết request xuyên nhiều service trong nền tảng SaaS B2B

**Hệ thống:** Một nền tảng SaaS quản lý dự án, một request tạo task đi qua service `api-gateway` → `task-service` → `notification-service` → `search-indexer` (bất đồng bộ).

**Vai trò của flow:** Distributed tracing dùng để nối các bước xử lý rải rác trên nhiều service thành một trace duy nhất, giúp debug khi một request "tạo task" bị chậm hoặc lỗi mà không rõ nghẽn ở đâu.

**Yêu cầu cụ thể:**
- Mỗi request phải được gắn một `trace_id` sinh ở gateway (nếu chưa có từ client) và propagate qua toàn bộ header của các lời gọi nội bộ tiếp theo, bao gồm cả lời gọi bất đồng bộ qua message queue.
- Mỗi service khi xử lý phải tạo một `span` con gắn với `trace_id` gốc và `parent_span_id` đúng, để dựng lại được cây gọi (call tree) đúng thứ tự và đúng quan hệ cha-con, kể cả khi các service gọi song song nhau.
- Xử lý được trường hợp một service không hỗ trợ tracing (legacy service bên thứ ba) — vẫn phải giữ được `trace_id` liên tục qua service đó (pass-through header) dù không có span chi tiết bên trong.
- Đảm bảo dữ liệu trace không làm rò rỉ thông tin nhạy cảm (không được gắn giá trị field chứa password/token vào tên span hay tag) — có allowlist rõ ràng cho các attribute được phép ghi vào trace.
- Tracing pipeline phải chịu được việc một service gửi trace data trễ hoặc mất gói (network drop) mà không làm sai lệch cây trace của các request khác — mỗi span phải tự đứng độc lập được, không phụ thuộc thứ tự nhận.
