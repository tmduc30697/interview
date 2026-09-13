# Ngân hàng số quản lý device trust cho giao dịch chuyển tiền giá trị lớn

**Hệ thống:** Một app ngân hàng số cho phép chuyển tiền, mở tài khoản tiết kiệm, và thực hiện các giao dịch tài chính giá trị lớn.

**Vai trò của flow:** Thiết bị mới phải qua xác minh bổ sung trước khi được thực hiện giao dịch giá trị cao, dù đã đăng nhập thành công bằng mật khẩu/MFA thông thường — tách biệt rõ "trust để đăng nhập xem thông tin" và "trust để thực hiện giao dịch lớn".

**Yêu cầu cụ thể:**
- Thiết bị đăng nhập lần đầu chỉ được cấp trust ở mức xem thông tin cơ bản; muốn chuyển khoản vượt một ngưỡng giá trị phải trải qua một cửa sổ "device maturity" (ví dụ thiết bị phải hoạt động ổn định qua một khoảng thời gian, hoặc hoàn tất xác minh bổ sung) trước khi được nâng lên mức trust cho phép giao dịch lớn.
- Phải chặn được kịch bản kẻ tấn công vừa chiếm được thông tin đăng nhập, đăng nhập trên thiết bị mới rồi cố chuyển tiền lớn ngay trong vài giây — tức là race condition giữa việc thiết lập trust thiết bị mới và việc thực hiện giao dịch giá trị cao gần như tức thời sau đó phải được xử lý an toàn theo hướng mặc định từ chối.
- Cần phân biệt hợp lý giữa "thiết bị thực sự mới lạ" và "thiết bị cũ bị cài lại app/xóa dữ liệu" của chính chủ tài khoản, để tránh làm phiền người dùng hợp lệ mỗi lần họ cài lại app trong khi vẫn giữ được mức bảo vệ cần thiết cho trường hợp thật sự đáng ngờ.
- Khi giao dịch bị chặn vì thiết bị chưa đủ trust, phải có luồng step-up rõ ràng và tức thời (ví dụ xác minh qua kênh độc lập như gọi điện, video call, hoặc sinh trắc học bổ sung) cho các trường hợp hợp lệ nhưng cần gấp, thay vì bắt người dùng chờ hàng giờ/ngày một cách cứng nhắc.
- Mọi thay đổi trust level của thiết bị (nâng lên, hạ xuống, revoke) phải có audit trail không thể chỉnh sửa, và phải cảnh báo ngay qua kênh độc lập (SMS/email) mỗi khi có thiết bị mới đạt được mức trust cho phép giao dịch lớn, để chủ tài khoản phát hiện sớm nếu không phải do chính họ thực hiện.
