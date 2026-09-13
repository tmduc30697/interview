# Tôn trọng rate limit của đối tác bên ngoài khi gửi SMS/OTP

**Hệ thống:** Nhiều service nội bộ (OTP đăng nhập, thông báo đơn hàng, marketing) cùng gọi tới một đối tác SMS gateway bên ngoài có giới hạn rate nghiêm ngặt (request/giây) áp dụng chung cho toàn bộ tài khoản đối tác.

**Vai trò của flow:** Throttling ở tầng gọi ra (outbound) đảm bảo tổng lưu lượng từ tất cả service nội bộ cộng lại không vượt giới hạn của đối tác, tránh bị đối tác chặn/phạt hoặc rớt message.

**Yêu cầu cụ thể:**
- Giới hạn của đối tác là giới hạn tổng (global) áp dụng chung cho toàn công ty, không phải cho riêng từng service — nếu mỗi service tự throttle độc lập theo giả định "được chia đều" thì tổng thực tế vẫn có thể vượt ngưỡng khi nhiều service cùng gửi cao điểm cùng lúc; cần một điểm throttle tập trung (hoặc counter chia sẻ) mà mọi service phải đi qua.
- Khi tổng lưu lượng vượt giới hạn đối tác cho phép tại một thời điểm, phải có cơ chế ưu tiên rõ ràng giữa các loại tin nhắn (OTP đăng nhập cần độ trễ thấp, ưu tiên hơn SMS marketing có thể trì hoãn vài phút), tránh để OTP bị xếp hàng chung với marketing gây trải nghiệm đăng nhập tệ.
- Khi bị đối tác trả về lỗi rate limit dù đã throttle, phải có backoff tăng dần và không được retry ngay lập tức theo kiểu spam lại, vì retry dồn dập có thể khiến đối tác giảm hạn mức hoặc tạm khóa tài khoản của cả công ty.
- Request bị throttle chờ tới lượt gửi phải có hàng đợi có TTL/hết hạn hợp lý (ví dụ OTP hết hạn nếu chờ quá vài chục giây thì bỏ, báo lỗi cho user thử lại) thay vì giữ trong queue vô thời hạn khiến user chờ OTP không tới nhưng cũng không được báo lỗi.
- Phải giám sát được usage thực tế so với hạn mức đối tác theo thời gian gần thực, cảnh báo sớm khi xu hướng traffic đang tiến gần ngưỡng (ví dụ do một service lỗi gửi lặp) trước khi bị đối tác chặn hoàn toàn ảnh hưởng tất cả service khác dùng chung gateway.
