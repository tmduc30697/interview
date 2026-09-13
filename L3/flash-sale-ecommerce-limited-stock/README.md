# Flash sale số lượng giới hạn trên sàn e-commerce (100 sản phẩm, hàng chục nghìn người chờ)

**Hệ thống:** Một sàn e-commerce tổ chức flash sale: đúng 12:00:00 mở bán 100 sản phẩm giá sốc, dự kiến hàng chục nghìn người cùng bấm "Mua ngay" trong vài giây đầu.

**Vai trò của flow:** Flow checkout phải đảm bảo đúng 100 người mua được (không hơn không kém), xử lý đúng khi hàng chục nghìn request cùng cạnh tranh vào đúng thời điểm mở bán, không để hệ thống sập hoặc bán vượt tồn kho (overselling).

**Yêu cầu cụ thể:**
- Không cho phép dùng `SELECT tồn_kho rồi kiểm tra > 0 rồi UPDATE trừ` ở tầng application (race condition kinh điển: nhiều request đọc thấy tồn kho > 0 cùng lúc rồi đều trừ) — yêu cầu dùng update nguyên tử kiểu `UPDATE stock SET qty = qty - 1 WHERE qty > 0` và kiểm tra số dòng bị ảnh hưởng (affected rows) để biết có mua được hay không.
- Thiết kế hàng đợi (queue) hoặc rate-limiter ở tầng trước DB để giảm áp lực: chỉ cho một số lượng request giới hạn/giây chạm vào transaction trừ tồn kho, các request còn lại nhận phản hồi "đang xử lý, vui lòng chờ" thay vì để tất cả cùng đấm vào DB gây timeout hàng loạt.
- Mô tả cụ thể: 2 request đến gần như đồng thời tại thời điểm tồn kho còn đúng 1 sản phẩm — hệ thống phải đảm bảo chỉ 1 request được xác nhận mua, request kia nhận lỗi "hết hàng" trong thời gian ngắn (không phải timeout), viết test giả lập 1000 request đồng thời cho 100 sản phẩm và assert đúng 100 request thành công.
- Idempotency: nếu client retry do timeout mạng (không rõ request trước có xử lý thành công hay không), request retry với cùng idempotency key không được trừ tồn kho lần 2 — phải trả lại đúng kết quả của lần xử lý đầu.
- Có cơ chế "circuit breaker"/giảm tải khi tồn kho đã về 0: các request đến sau phải được chặn ngay từ tầng cache/API gateway, không để lọt xuống DB gây tải vô ích.
