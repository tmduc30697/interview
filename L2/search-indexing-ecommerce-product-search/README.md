# E-commerce tìm kiếm và gợi ý sản phẩm

**Hệ thống:** Một sàn e-commerce với hàng triệu sản phẩm, cần thanh tìm kiếm với autocomplete và kết quả tìm kiếm chính xác theo tên/thuộc tính sản phẩm.

**Vai trò của flow:** Đồng bộ dữ liệu sản phẩm (tên, giá, tồn kho, danh mục) từ database chính vào search index, và phục vụ autocomplete/tìm kiếm với độ trễ thấp.

**Yêu cầu cụ thể:**
- Khi sản phẩm thay đổi giá hoặc hết hàng, search index phải được cập nhật trong vòng vài giây, tránh hiển thị giá cũ hoặc cho phép tìm thấy sản phẩm đã hết hàng mà không có cảnh báo rõ ràng.
- Autocomplete phải trả kết quả trong thời gian rất ngắn (dưới 100ms) và ưu tiên gợi ý sản phẩm đang bán tốt/còn hàng lên trước, không chỉ khớp chuỗi ký tự đơn thuần.
- Index phải xử lý được lỗi chính tả và từ đồng nghĩa phổ biến (ví dụ "tivi"/"tv", dấu tiếng Việt có/không dấu) để không bỏ lỡ kết quả đúng khi người dùng gõ sai/thiếu dấu.
- Khi có cập nhật hàng loạt (bulk update giá dịp sale lớn ảnh hưởng hàng trăm nghìn sản phẩm), pipeline đồng bộ vào index không được làm chậm hoặc gây downtime cho tìm kiếm đang phục vụ người dùng thật.
- Có cơ chế phát hiện và tự phục hồi khi index bị lệch so với database nguồn (ví dụ do lỗi đồng bộ), thông qua job đối soát định kỳ chạy full reindex một phần hoặc toàn phần mà không ảnh hưởng tìm kiếm đang chạy (dùng index song song rồi swap).
