# Marketplace e-commerce phát hiện session bất thường và yêu cầu step-up

**Hệ thống:** Một sàn thương mại điện tử cho phép người dùng lưu thông tin thanh toán và địa chỉ để mua hàng nhanh.

**Vai trò của flow:** Quản lý session "remember me" thông thường cho trải nghiệm mua sắm hằng ngày, nhưng phát hiện được dấu hiệu session bị chiếm quyền (session hijacking) để yêu cầu xác thực bổ sung đúng lúc mà không làm phiền người dùng hợp lệ.

**Yêu cầu cụ thể:**
- Session "remember me" (giữ đăng nhập lâu dài) phải được phân tách với các hành động nhạy cảm (thanh toán, đổi địa chỉ giao hàng, đổi phương thức thanh toán) — những hành động này yêu cầu xác thực bổ sung dù session chính vẫn còn hạn.
- Hệ thống phải theo dõi tín hiệu bất thường trong phiên (thay đổi IP/vị trí địa lý giữa các request liên tiếp một cách bất hợp lý, thay đổi user-agent giữa chừng) để gắn cờ session nghi ngờ bị đánh cắp.
- Khi một session bị gắn cờ nghi ngờ, hệ thống phải yêu cầu xác thực lại trước khi cho tiếp tục các hành động nhạy cảm, đồng thời không tự động đăng xuất ngay lập tức nếu chỉ đang xem sản phẩm (tránh làm phiền người dùng hợp lệ vì false positive).
- Khi phát hiện thanh toán được thực hiện ngay sau một thay đổi thông tin nhạy cảm gần đây (ví dụ vừa đổi địa chỉ giao hàng/phương thức thanh toán), phải tự động yêu cầu xác thực lại trước khi cho hoàn tất giao dịch, để chặn kịch bản chiếm session rồi đổi địa chỉ nhận hàng.
- Cung cấp cho người dùng lịch sử các lần đăng nhập/session gần đây (thiết bị, vị trí ước lượng, thời gian) và cách báo cáo nếu phát hiện session không phải của họ, kèm hành động tự động khóa tạm giao dịch đang treo khi người dùng báo cáo bị chiếm session.
