# Marketplace đa seller đối soát payout cho người bán

**Hệ thống:** Một marketplace đa seller, sàn thu tiền từ người mua rồi định kỳ trả (payout) cho từng seller sau khi trừ phí hoa hồng sàn và các khoản hoàn tiền phát sinh.

**Vai trò của flow:** Đối soát giữa số tiền hệ thống nội bộ tính seller được nhận (sau phí, sau hoàn tiền) và số tiền thực tế đã được chuyển ra ngoài qua cổng thanh toán cho từng seller.

**Yêu cầu cụ thể:**
- Một seller có thể có hàng trăm đơn hàng hoàn tất trong kỳ, một số đơn bị hoàn tiền/khiếu nại xử lý sau khi đã tính vào kỳ payout — cần xác định rõ mốc thời gian cắt (cutoff) nhất quán cho từng kỳ, tránh trường hợp đơn vừa hoàn tất bị tính hai lần vào hai kỳ khác nhau hoặc bị bỏ sót không kỳ nào tính tới.
- Khi một đơn hàng đã được tính vào payout đã chuyển tiền cho seller, nhưng sau đó phát sinh hoàn tiền cho người mua, hệ thống phải xử lý đúng việc thu hồi/trừ vào kỳ payout tiếp theo của seller đó — đối soát phải phát hiện được các khoản âm này thay vì chỉ xử lý cộng dồn một chiều.
- Lệnh chuyển tiền ra ngoài cho seller qua cổng thanh toán có thể ở trạng thái pending/thất bại/bị trả lại (ví dụ tài khoản ngân hàng seller sai thông tin) — đối soát phải phân biệt rõ "đã lên lệnh chuyển" và "seller thực nhận được tiền", không đóng kỳ payout khi còn lệnh chuyển chưa xác nhận thành công.
- Phí hoa hồng sàn được tính tại thời điểm đơn hàng hoàn tất có thể khác với mức phí áp dụng tại thời điểm đối soát nếu chính sách phí thay đổi giữa chừng — đối soát phải dùng đúng mức phí đã chốt tại thời điểm phát sinh giao dịch, không tính lại theo cấu hình phí hiện tại gây sai lệch số liệu lịch sử.
- Khi phát hiện sai lệch giữa số tiền sàn tính seller được nhận và số tiền thực đã chuyển, cần có cơ chế giữ (hold) kỳ payout tiếp theo của seller đó lại để điều chỉnh, thay vì tiếp tục chuyển tiền theo số liệu có khả năng sai, đồng thời không làm ảnh hưởng payout của các seller khác không liên quan.
