# Nền tảng bất động sản tìm kiếm theo bản đồ và autocomplete địa chỉ

**Hệ thống:** Một nền tảng đăng tin bất động sản cho phép người tìm mua/thuê tra cứu theo khu vực trên bản đồ kết hợp autocomplete gợi ý địa chỉ/khu vực.

**Vai trò của flow:** Lập chỉ mục tin bất động sản kèm dữ liệu địa lý (tọa độ, ranh giới khu vực), phục vụ tìm kiếm theo vùng bản đồ và autocomplete địa chỉ chính xác.

**Yêu cầu cụ thể:**
- Tìm kiếm theo vùng hiển thị trên bản đồ (bounding box/bán kính từ một điểm) phải trả đúng và đủ các tin nằm trong vùng đó, xử lý chính xác các trường hợp tin ở biên giới vùng (gần đúng ranh giới quận/khu vực).
- Autocomplete địa chỉ phải xử lý được cách viết địa chỉ không thống nhất (viết tắt, thiếu dấu, thứ tự phường/quận/thành phố khác nhau) và gợi ý đúng địa chỉ chuẩn hóa, không yêu cầu người dùng gõ chính xác định dạng địa chỉ đầy đủ.
- Khi người đăng tin cập nhật tọa độ/vị trí của bất động sản (ví dụ sửa lại vị trí đánh dấu trên bản đồ cho chính xác hơn), index phải cập nhật ngay để tin xuất hiện đúng ở kết quả tìm kiếm theo vùng bản đồ mới, biến mất khỏi vùng cũ.
- Tin đã cho thuê/bán xong phải được gỡ khỏi kết quả tìm kiếm theo bản đồ ngay lập tức, tránh người tìm liên hệ vào tin không còn hiệu lực gây trải nghiệm xấu.
- Khi người dùng zoom/pan bản đồ liên tục để khám sát khu vực, hệ thống phải trả kết quả tìm kiếm theo vùng mới với độ trễ thấp cho từng lần thay đổi vùng nhìn, không tính toán lại toàn bộ từ đầu mỗi lần một cách tốn kém tài nguyên.
