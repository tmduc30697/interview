# E-commerce tối ưu query lọc & sắp xếp catalog sản phẩm nhiều điều kiện

**Hệ thống:** Sàn e-commerce có catalog hàng triệu sản phẩm, trang danh mục cho phép user kết hợp nhiều bộ lọc (category, brand, khoảng giá, rating, còn hàng) cùng lúc với nhiều kiểu sắp xếp (giá tăng/giảm, bán chạy, mới nhất).

**Vai trò của flow:** Chọn đúng loại index (composite/covering/partial) cho service catalog để trả kết quả nhanh dưới nhiều tổ hợp filter/sort khác nhau, mà không tạo tràn lan index thừa làm chậm ghi và phình dung lượng.

**Yêu cầu cụ thể:**
- Composite index cho tổ hợp category + brand + price hoạt động tốt khi ORDER BY đúng theo thứ tự cột cuối của index, nhưng khi user đổi sang sort theo tiêu chí khác (bán chạy, rating) không nằm trong index đó, DB phải filesort bổ sung — cần quyết định rõ tổ hợp sort nào được index hỗ trợ trực tiếp và tổ hợp nào chấp nhận chậm hơn.
- Số tổ hợp filter user có thể chọn là rất lớn (category × brand × price range × rating), không thể tạo index cho mọi tổ hợp — phải dựa trên phân tích tần suất sử dụng thực tế để chọn ra vài composite index phổ biến nhất, và có chiến lược fallback (query chậm hơn, hoặc đẩy sang search engine riêng) cho tổ hợp hiếm.
- Giá và tồn kho trên bảng products được cập nhật rất thường xuyên (real-time, nhiều lần/giây ở sản phẩm hot) — mỗi index thêm vào đều làm chậm write này, cần cân bằng rõ giữa số lượng index phục vụ đọc và overhead ghi.
- Khi user gõ từ khóa tìm kiếm tên sản phẩm kết hợp với filter (vd "áo thun nam" + lọc giá), B-tree index thường không đáp ứng tốt free-text search — cần xác định ranh giới khi nào dùng index DB thường, khi nào phải tách sang search index riêng (Elasticsearch-like).
- Khi user phân trang sâu (page 200+) bằng OFFSET, DB phải quét và bỏ qua toàn bộ các dòng phía trước dù có index, khiến độ trễ tăng dần theo số trang — cần chuyển sang keyset pagination (dựa trên giá trị cột index cuối cùng đã thấy) để giữ tốc độ ổn định bất kể phân trang sâu tới đâu.
