# SaaS thuê bao đối soát billing nội bộ với payment processor

**Hệ thống:** Một SaaS thu phí thuê bao định kỳ hàng tháng, khách hàng có thể nâng/hạ cấp gói giữa chu kỳ, thanh toán xử lý qua một payment processor bên ngoài.

**Vai trò của flow:** Đối soát giữa hệ thống billing nội bộ (invoice phát sinh, các khoản điều chỉnh proration khi đổi gói) và số tiền thực tế processor đã thu được/báo cáo lại theo từng chu kỳ thanh toán.

**Yêu cầu cụ thể:**
- Khi khách hàng đổi gói giữa chu kỳ, hệ thống billing nội bộ phát sinh nhiều invoice điều chỉnh nhỏ (giảm trừ phần chưa dùng của gói cũ, cộng thêm phần gói mới) trong cùng một khoảng thời gian ngắn — đối soát phải khớp đúng tổng các điều chỉnh này với đúng một lượt thu tiền tương ứng từ processor, tránh hiểu nhầm là nhiều giao dịch riêng lẻ không khớp.
- Processor có thể thử thu tiền thất bại rồi tự động retry theo lịch riêng của họ (dunning) trong vài ngày sau ngày đến hạn — đối soát phải xử lý được trạng thái "đang chờ retry" khác với "thất bại hẳn", không đóng sổ chu kỳ khi khoản thu vẫn còn khả năng thành công ở lần retry tiếp theo.
- Khách hàng hủy gói giữa chu kỳ có thể được hoàn lại phần tiền chưa dùng, khoản hoàn này phát sinh sau khi invoice gốc đã đối soát khớp từ trước — cần cập nhật lại trạng thái đối soát của invoice đó, không để hệ thống báo cáo doanh thu vẫn tính khoản đã hoàn là doanh thu thực.
- Số tiền processor thực báo cáo có thể đã trừ phí xử lý giao dịch trước khi ghi có, trong khi hệ thống billing nội bộ ghi nhận theo giá trị gộp trước phí — đối soát phải tách rõ hai luồng số liệu này để không nhầm chênh lệch phí xử lý thành sai lệch thực sự cần điều tra.
- Khi múi giờ đóng chu kỳ billing nội bộ không trùng khớp với thời điểm processor tổng hợp báo cáo theo lịch của họ, một số giao dịch cận ranh giới ngày có thể bị xếp lệch chu kỳ giữa hai bên — cần định nghĩa rõ quy tắc xếp chu kỳ thống nhất và có bước xử lý riêng cho các giao dịch cận biên này thay vì báo sai lệch giả mỗi tháng.
