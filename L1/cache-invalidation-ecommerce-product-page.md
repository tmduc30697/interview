# Cache trang chi tiết sản phẩm cho e-commerce

**Hệ thống:** Trang e-commerce cache HTML/dữ liệu trang chi tiết sản phẩm (giá, tồn kho, mô tả) để giảm tải DB cho lượng truy cập lớn.

**Vai trò của flow:** Invalidate đúng cache khi sản phẩm được cập nhật (đổi giá, hết hàng) để không hiển thị thông tin sai cho khách, đồng thời giữ hiệu quả cache cao khi sản phẩm không đổi.

**Yêu cầu cụ thể:**
- Khi giá hoặc tồn kho sản phẩm thay đổi ở hệ thống quản trị, cache liên quan phải được invalidate trong vòng vài giây, không dựa hoàn toàn vào TTL cố định dài (ví dụ TTL 1 giờ là không đủ nhanh cho thay đổi giá).
- Có chiến lược rõ giữa cache invalidation theo event (khi có thay đổi, chủ động xóa) và TTL ngắn dự phòng (để tránh trường hợp event bị lỡ vẫn tự hết hạn sau một khoảng thời gian giới hạn).
- Xử lý race condition: request đọc cache đúng lúc dữ liệu đang được invalidate/update không được thấy trạng thái nửa cũ nửa mới (ví dụ giá mới nhưng tồn kho cũ).
- Với sản phẩm có traffic rất cao (best-seller), invalidate không được gây "cache stampede" (mọi request cùng lúc miss cache dội xuống DB) — cần cơ chế warm lại cache có kiểm soát (single flight hoặc pre-warm).
- Đo lường: cache hit ratio theo loại trang, và độ trễ giữa lúc dữ liệu gốc đổi tới lúc cache phản ánh đúng (staleness window thực tế).
