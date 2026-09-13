# Cách ly dữ liệu bán hàng giữa các seller trên marketplace dùng chung nền tảng

**Hệ thống:** Một marketplace cho phép nhiều người bán (seller) độc lập đăng sản phẩm và bán hàng trên cùng một nền tảng dùng chung hạ tầng, mỗi seller có dữ liệu đơn hàng, doanh thu, và thông tin khách hàng của riêng họ.

**Vai trò của flow:** Cách ly ở đây phải đảm bảo một seller không bao giờ nhìn thấy dữ liệu bán hàng/doanh thu của seller khác dù họ đang cùng thao tác trên chung một giao diện quản lý (dashboard người bán) và chung hạ tầng backend, khác với mô hình một công ty một tenant thông thường vì số lượng seller có thể lên tới hàng chục nghìn và liên tục có seller mới tham gia/rời đi.

**Yêu cầu cụ thể:**
- Mọi truy vấn liên quan tới đơn hàng, doanh thu, tồn kho trên dashboard người bán phải bị giới hạn đúng theo `seller_id` ở tầng thấp nhất (không dựa vào việc ẩn nút bấm trên giao diện), kể cả với các API nội bộ dùng chung để tính toán số liệu tổng hợp cho nhiều mục đích khác nhau của nền tảng.
- Xử lý đúng trường hợp một đơn hàng có sản phẩm từ nhiều seller khác nhau trong cùng một giỏ hàng của khách mua — mỗi seller chỉ được xem phần đơn hàng liên quan tới sản phẩm của mình (thông tin vận chuyển, số lượng, giá của phần họ bán), không được thấy toàn bộ chi tiết đơn hàng gộp bao gồm phần của seller khác.
- Đảm bảo các báo cáo/thống kê xếp hạng do nền tảng cung cấp cho seller (ví dụ so sánh hiệu suất bán hàng với "mức trung bình ngành") không bị suy luận ngược ra số liệu cụ thể của một seller đối thủ cụ thể — đặc biệt khi ngành hàng đó chỉ có rất ít seller tham gia khiến số liệu trung bình gần như lộ số liệu của một seller cụ thể.
- Xử lý trường hợp một seller ngừng hoạt động/bị khóa tài khoản (vi phạm chính sách hoặc tự nguyện rời sàn) — dữ liệu lịch sử đơn hàng liên quan tới khách mua vẫn cần được giữ cho mục đích hỗ trợ khách hàng và pháp lý, nhưng seller đã rời sàn không được tiếp tục truy cập vào dữ liệu đó, và các seller khác cũng không được vô tình thấy dữ liệu của seller đã rời sàn qua các báo cáo tổng hợp toàn nền tảng.
- Đảm bảo tính năng chăm sóc khách hàng của nền tảng (nhân viên marketplace hỗ trợ khách mua) khi cần tra cứu một đơn hàng cụ thể chỉ được xem đúng phạm vi dữ liệu liên quan tới đơn hàng đó, không được có quyền truy cập mặc định vào toàn bộ dữ liệu kinh doanh của seller liên quan, và mọi lần tra cứu phải được ghi log lại.
