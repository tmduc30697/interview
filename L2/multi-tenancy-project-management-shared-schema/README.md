# Cách ly dữ liệu theo tenant dùng chung schema/database cho SaaS quản lý dự án

**Hệ thống:** Một nền tảng SaaS quản lý dự án cho nhiều công ty khách hàng (tenant), dùng chung một database và schema, phân biệt dữ liệu bằng cột `tenant_id` trên mỗi bảng.

**Vai trò của flow:** Đây là mô hình multi-tenancy phổ biến nhất (shared schema) — flow phải đảm bảo mọi truy vấn dữ liệu đều bị giới hạn đúng theo tenant của người dùng đang thao tác, không có đường nào truy cập nhầm sang dữ liệu tenant khác.

**Yêu cầu cụ thể:**
- Mọi query đọc/ghi dữ liệu phải bắt buộc có điều kiện lọc theo `tenant_id` ở tầng thấp nhất có thể (ví dụ qua middleware/ORM tự động thêm điều kiện, hoặc row-level security ở database) — không phụ thuộc vào việc lập trình viên nhớ thêm điều kiện này ở từng câu query riêng lẻ, vì chỉ cần quên một chỗ là có thể lộ dữ liệu chéo tenant.
- Viết test tự động (không chỉ manual test) cố tình thử truy cập dữ liệu của tenant B bằng token/session của tenant A qua mọi endpoint quan trọng, đảm bảo luôn bị chặn (403/404) không phải chỉ dựa vào việc UI không hiển thị link dẫn tới đó.
- Xử lý đúng các tài nguyên dùng chung giữa nhiều tenant nhưng cần cấu hình riêng (ví dụ file upload, cấu hình thông báo) — đảm bảo đường dẫn lưu trữ và khóa truy cập (như signed URL) đều gắn chặt với tenant, không dùng ID tài nguyên có thể đoán được để truy cập chéo tenant.
- Xử lý trường hợp một nhân viên hệ thống (support/admin nội bộ) cần truy cập dữ liệu của một tenant cụ thể để hỗ trợ — phải qua một cơ chế impersonation/xem hộ có kiểm soát, ghi log đầy đủ, không dùng chung quyền truy cập không giới hạn tenant cho mục đích vận hành.
- Đảm bảo các thao tác nền (background job, cron, cache) cũng tôn trọng ranh giới tenant — ví dụ một job tính báo cáo tổng hợp chạy sai logic gộp nhầm dữ liệu của nhiều tenant vào một kết quả chung phải được phát hiện qua test, không chỉ API request trực tiếp mới cần kiểm tra.
