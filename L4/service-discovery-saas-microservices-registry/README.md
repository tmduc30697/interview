# Service registry cho nền tảng SaaS microservices

**Hệ thống:** Một nền tảng SaaS gồm khoảng 10 microservice (auth, billing, notification, ...) giao tiếp qua HTTP nội bộ, cần một registry trung tâm để các service tìm nhau.

**Vai trò của flow:** Mỗi service tự đăng ký khi khởi động và bị loại khỏi registry khi không còn healthy, để các service khác luôn gọi tới đúng instance đang sống.

**Yêu cầu cụ thể:**
- Mỗi instance khi start phải tự đăng ký vào registry kèm địa chỉ, port, version, và metadata (ví dụ zone/datacenter); khi nhận được shutdown signal phải tự deregister trước khi tắt.
- Registry phải phân biệt được health check "active" (registry tự ping endpoint `/health` định kỳ) và "passive" (instance tự gửi heartbeat) — chọn một chiến lược và giải thích trade-off, xử lý được trường hợp instance vẫn sống nhưng mạng tới registry bị nghẽn tạm thời (không được xóa nhầm).
- Nếu một instance crash mà không kịp deregister (kill -9, mất điện), registry phải tự phát hiện qua TTL/heartbeat timeout và loại nó ra trong một khoảng thời gian xác định (SLA phát hiện, ví dụ dưới 30 giây).
- Client (service gọi) phải cache danh sách instance để giảm tải cho registry, nhưng vẫn phải cập nhật kịp thời khi có thay đổi — định nghĩa rõ TTL cache và cơ chế invalidate.
- Xử lý race condition: hai instance cùng đăng ký gần như đồng thời với cùng service name nhưng version khác nhau trong lúc đang rolling deploy — client gọi phải nhận được danh sách nhất quán, không bị lẫn instance cũ/mới theo cách gây lỗi.
