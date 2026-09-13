# Nền tảng tuyển dụng với yêu cầu độ tươi mới của tin đăng

**Hệ thống:** Một nền tảng tuyển dụng cho phép ứng viên tìm việc theo kỹ năng/địa điểm/mức lương, nhà tuyển dụng đăng tin và có thể đóng tin bất cứ lúc nào khi đã tuyển đủ.

**Vai trò của flow:** Lập chỉ mục tin tuyển dụng với các thuộc tính lọc (kỹ năng, địa điểm, mức lương), phục vụ tìm kiếm và đảm bảo tin hết hạn/đã tuyển đủ biến mất khỏi kết quả ngay lập tức.

**Yêu cầu cụ thể:**
- Khi nhà tuyển dụng đánh dấu tin đã tuyển đủ hoặc chủ động đóng tin, tin đó phải biến mất khỏi kết quả tìm kiếm ngay lập tức, không chờ tới chu kỳ đồng bộ định kỳ — vì ứng viên nộp hồ sơ vào tin đã đóng gây trải nghiệm xấu và tổn hại uy tín nền tảng nhiều hơn so với các loại nội dung khác chấp nhận độ trễ index cao hơn.
- Tin tuyển dụng có ngày hết hạn tự động phải được gỡ khỏi index đúng thời điểm hết hạn mà không cần nhà tuyển dụng chủ động thao tác, xử lý đúng múi giờ và không để lệch vài giờ khiến tin hết hạn vẫn hiển thị hoặc tin còn hạn bị gỡ nhầm.
- Tìm kiếm theo mức lương phải xử lý được các tin để khoảng lương (min-max) thay vì một con số cố định, và một số tin không công khai mức lương — bộ lọc phải trả kết quả đúng logic giao khoảng khi ứng viên lọc theo một mức lương mong muốn cụ thể, không loại nhầm các tin có khoảng lương chỉ giao một phần với tiêu chí lọc.
- Kỹ năng yêu cầu trong tin đăng thường được nhà tuyển dụng gõ tự do, trong khi ứng viên tìm theo tên kỹ năng chuẩn — autocomplete và index phải ánh xạ được các cách viết/tên gọi khác nhau của cùng một kỹ năng (viết tắt, đồng nghĩa) về cùng một khái niệm để không bỏ lỡ tin phù hợp chỉ vì khác cách gõ.
- Khi một công ty đăng số lượng lớn tin cùng lúc hoặc gỡ hàng loạt tin cùng lúc, pipeline index phải xử lý được đợt thay đổi hàng loạt này mà không tạo ra độ trễ lan sang việc cập nhật index cho các tin đăng bình thường khác của nhà tuyển dụng khác đang diễn ra song song.
