# Tạo vận đơn qua API đối tác vận chuyển (shipping carrier)

**Hệ thống:** Hệ thống logistics/e-commerce gọi API của nhiều đối tác vận chuyển (carrier) bên ngoài để tạo vận đơn, tra cứu trạng thái giao hàng, mỗi đối tác có SLA và độ ổn định khác nhau, đặc biệt kém vào giờ cao điểm.

**Vai trò của flow:** Circuit breaker riêng biệt cho từng đối tác vận chuyển (không dùng chung một breaker cho tất cả) để lỗi của một carrier không ảnh hưởng luồng gọi carrier khác, kèm fallback tự động chuyển đơn sang đối tác thay thế khi carrier chính đang gặp sự cố.

**Yêu cầu cụ thể:**
- Mỗi đối tác vận chuyển phải có circuit breaker độc lập với ngưỡng riêng (có thể khác nhau theo lịch sử SLA từng đối tác) — tuyệt đối không dùng chung một breaker toàn cục, vì như vậy một carrier bị lỗi nặng sẽ khóa luôn cả các carrier khác đang hoạt động bình thường.
- Khi breaker của carrier A mở, hệ thống phải tự động fallback sang carrier B/C theo thứ tự ưu tiên đã cấu hình (dựa trên vùng phục vụ, chi phí, hoặc SLA hiện tại), nhưng phải tránh trường hợp đơn hàng đã thực sự được tạo phía carrier A (do timeout khi đang chờ phản hồi, không phải lỗi thực sự) rồi lại bị tạo trùng lần nữa ở carrier B.
- Với API tạo vận đơn (ghi, có tác dụng phụ thật ngoài đời — carrier đã cấp mã vận đơn và lên lịch lấy hàng), retry chỉ được thực hiện sau khi xác minh qua API tra cứu trạng thái của chính carrier đó rằng vận đơn thực sự chưa được tạo, không retry mù dựa trên timeout đơn thuần.
- Giờ cao điểm là lúc carrier dễ chậm/lỗi nhất nhưng cũng là lúc lượng đơn cần xử lý cao nhất — backoff giữa các lần retry phải có jitter để tránh hàng loạt request retry cùng dồn vào carrier đúng lúc carrier đang phục hồi, càng làm carrier chậm phục hồi hơn.
- Đo lường riêng theo từng carrier: tỉ lệ lỗi, thời gian breaker mở trong ngày, tỉ lệ đơn phải fallback sang carrier dự phòng, và chi phí phát sinh do phải chuyển sang carrier dự phòng có giá cao hơn — dùng để đánh giá định kỳ có nên thay đổi thứ tự ưu tiên carrier hay không.
