# Trace lời gọi từ mobile app xuyên qua backend tới các dịch vụ bên thứ ba

**Hệ thống:** Một ứng dụng mobile gọi vào backend, và với một số tính năng, backend phải gọi tiếp ra các dịch vụ bên thứ ba (bản đồ/định vị, gửi SMS xác thực, đẩy push notification) trước khi trả kết quả về cho app.

**Vai trò của flow:** Tracing phải phân biệt rõ ràng thời gian chờ các dịch vụ bên thứ ba (ngoài tầm kiểm soát của đội kỹ thuật) với thời gian backend tự xử lý nội bộ, để khi app "cảm giác chậm", đội vận hành biết nên gây sức ép với đối tác thứ ba hay tối ưu code nội bộ.

**Yêu cầu cụ thể:**
- Mỗi lời gọi ra dịch vụ bên thứ ba phải được bọc trong một span riêng, gắn tag rõ ràng là external call kèm tên nhà cung cấp, tách biệt hoàn toàn khỏi span xử lý logic nội bộ, để khi tổng hợp latency trung bình của một API, có thể bóc tách được phần "chờ bên ngoài" ra khỏi phần "backend tự xử lý".
- Vì kết nối từ mobile app tới backend qua mạng di động không ổn định, phải phân biệt được độ trễ do chính mạng di động của client (trước khi request tới được backend) với độ trễ phát sinh trong nội bộ hệ thống backend — tránh gộp chung khiến số liệu latency nội bộ bị nhiễu bởi yếu tố phía client hoàn toàn không kiểm soát được.
- Xử lý trường hợp một dịch vụ bên thứ ba (ví dụ SMS) timeout hoặc trả lỗi và backend có cơ chế retry — mỗi lần retry phải tạo span riêng biệt kèm số thứ tự lần thử, để phân biệt được "một cuộc gọi chậm" với "ba lần gọi cộng dồn lại thành chậm", vốn cần xử lý khác nhau về mặt vận hành.
- Thiết kế cảnh báo riêng theo từng nhà cung cấp bên thứ ba (SLA latency/tỷ lệ lỗi riêng cho bản đồ, SMS, push notification) để phát hiện khi một đối tác cụ thể đang suy giảm chất lượng dịch vụ, thay vì chỉ có một cảnh báo chung chung "API chậm" gộp tất cả nguyên nhân lại.
- Đảm bảo dữ liệu trace không vô tình ghi lại nội dung nhạy cảm được gửi qua các dịch vụ bên thứ ba (ví dụ nội dung tin nhắn SMS chứa mã OTP, tọa độ vị trí chính xác của người dùng) vào tag/log của span — chỉ ghi metadata cần thiết cho việc debug hiệu năng (thời gian, trạng thái, mã lỗi), không ghi nội dung payload thật.
