# Chống spam tin nhắn trong app chat/group

**Hệ thống:** Ứng dụng chat/group cho phép user gửi tin nhắn tự do, cần chặn bot/spam gửi hàng loạt tin nhắn mà không làm phiền user thật đang gõ nhanh trong cuộc trò chuyện sôi nổi.

**Vai trò của flow:** Rate limiting theo user/theo cuộc trò chuyện giới hạn tần suất gửi tin để phát hiện và chặn hành vi spam, đồng thời đủ khoan dung cho tốc độ gõ/gửi tự nhiên của người dùng bình thường.

**Yêu cầu cụ thể:**
- Ngưỡng rate limit cứng (ví dụ giới hạn tin/giây cố định) dễ chặn nhầm user thật đang nhắn tin dồn dập trong lúc trò chuyện sôi nổi — cần thuật toán cho phép burst ngắn tự nhiên (vài tin liên tiếp trong 1-2 giây) nhưng vẫn bắt được pattern đều đặn máy móc kéo dài của bot.
- Phân biệt rate limit theo từng cuộc trò chuyện một-một và theo group lớn (hàng trăm/nghìn thành viên) — cùng một user gửi nhanh trong group đông có thể là bình thường (đang thảo luận sôi nổi), trong khi cùng tần suất đó nhắm vào nhiều user khác nhau (spam hàng loạt) phải bị đánh dấu khác.
- Khi user bị giới hạn do gửi quá nhanh, phản hồi (ẩn tạm nút gửi, thông báo chờ vài giây) không được làm mất nội dung tin nhắn đang gõ dở — user không được yêu cầu gõ lại từ đầu chỉ vì bị rate limit tạm thời.
- Phát hiện spam không chỉ dựa vào tần suất mà còn pattern nội dung lặp lại (cùng một tin gửi liên tục tới nhiều group/user khác nhau trong thời gian ngắn) — một tài khoản gửi đúng nhịp một tin/giây liên tục hàng giờ tới nhiều đối tượng khác nhau là dấu hiệu bot dù tần suất từng tin không vượt ngưỡng cứng.
- Rate limit phải áp dụng được động theo uy tín tài khoản (tài khoản mới tạo/chưa xác thực bị giới hạn chặt hơn tài khoản lâu năm có lịch sử hành vi tốt), tránh chặn quá nghiêm với user thật lâu năm trong khi vẫn kiểm soát chặt tài khoản mới dễ bị lợi dụng để spam.
