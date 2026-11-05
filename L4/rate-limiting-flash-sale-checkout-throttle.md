# Throttling request checkout trong flash sale để bảo vệ hệ thống backend

**Hệ thống:** E-commerce cần throttle lượng request checkout đồng thời trong giờ flash sale để tránh backend (DB, payment) bị quá tải sập hoàn toàn.

**Vai trò của flow:** Giới hạn số lượng request được xử lý đồng thời ở tầng vào (ví dụ qua hàng đợi ảo/virtual waiting room) để bảo vệ hệ thống, đổi lại một số user phải chờ theo lượt.

**Yêu cầu cụ thể:**
- Khi lượng request checkout vượt khả năng xử lý an toàn của backend, hệ thống phải chuyển user vào "phòng chờ" (queue) hiển thị vị trí/thời gian ước tính, không để request dồn thẳng vào backend gây sập.
- Thứ tự vào phòng chờ phải công bằng (FIFO theo thời điểm request tới) trừ khi có chính sách ưu tiên rõ ràng được công bố trước (ví dụ khách VIP), tránh cảm giác không công bằng cho user thường.
- Hệ thống throttle phải điều chỉnh động số lượng cho qua dựa trên tình trạng thực tế của backend (ví dụ giảm số cho qua nếu latency DB tăng cao), không dùng số cố định tĩnh không phản ánh tải thật.
- Đảm bảo user đã vào được luồng xử lý (qua khỏi hàng chờ) không bị rate limit lần nữa giữa đường gây trải nghiệm xấu (đã chờ xong lại bị chặn tiếp).
- Đo lường và báo cáo: throughput checkout thành công/giây tối đa hệ thống đạt được an toàn, và tỉ lệ user phải vào hàng chờ so với tổng traffic trong giờ flash sale, dùng để lên kế hoạch capacity cho các sự kiện sau.
