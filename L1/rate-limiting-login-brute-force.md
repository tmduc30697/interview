# Chống brute-force ở endpoint đăng nhập

**Hệ thống:** Một web app cần bảo vệ endpoint login khỏi tấn công dò mật khẩu (brute force/credential stuffing).

**Vai trò của flow:** Rate limiting theo nhiều chiều (theo IP, theo username, theo device) để làm chậm/ngăn tấn công dò mật khẩu mà không gây khó cho user hợp lệ gõ sai vài lần.

**Yêu cầu cụ thể:**
- Giới hạn số lần login sai theo username cụ thể (ví dụ 5 lần/15 phút) độc lập với giới hạn theo IP (ví dụ 20 lần/15 phút toàn bộ username khác nhau từ 1 IP) để bắt được cả 2 kiểu tấn công (dò 1 tài khoản nhiều lần, và dò nhiều tài khoản từ 1 nguồn).
- Sau khi vượt ngưỡng, phải có cơ chế tăng dần độ khó (progressive delay/CAPTCHA) trước khi chặn hoàn toàn, tránh chặn cứng ngay gây trải nghiệm xấu cho user thật chỉ gõ nhầm vài lần.
- Rate limit không được dựa hoàn toàn vào IP đơn lẻ vì tấn công thực tế thường dùng nhiều IP (botnet) — phải kết hợp thêm fingerprint thiết bị hoặc pattern hành vi bất thường.
- Login thành công phải reset counter rate limit của username đó, không giữ user hợp lệ trong trạng thái gần bị khóa do vài lần gõ sai trước đó.
- Log và alert khi phát hiện pattern tấn công rõ ràng (số lượng lớn login fail trong thời gian ngắn từ nhiều IP nhắm vào nhiều username) để team security theo dõi real-time.
