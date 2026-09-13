# Đổi đơn vị tiền tệ lưu trữ từ float sang integer (cents) trong hệ thống fintech

**Hệ thống:** Một app quản lý chi tiêu cá nhân đang lưu số tiền dạng `float` (dễ sai số làm tròn), cần migrate toàn bộ sang lưu `integer` (đơn vị cents) mà không làm sai lệch số dư của user nào trong lúc migrate.

**Vai trò của flow:** Flow dual-write phải đảm bảo mọi giao dịch mới trong lúc migrate được ghi đúng ở cả cột `amount_float` (cũ) và `amount_cents` (mới), với công thức chuyển đổi nhất quán, để không tạo ra sai số dồn tích ảnh hưởng đến số dư hiển thị cho user.

**Yêu cầu cụ thể:**
- Quy định công thức chuyển đổi cụ thể (ví dụ round-half-up ở đơn vị cents) và bắt buộc áp dụng đồng nhất ở cả chiều ghi mới và chiều backfill dữ liệu cũ, để tránh 2 cách round khác nhau tạo lệch số dư giữa 2 cột.
- Mô tả race condition: giao dịch A (ghi tiền vào) và giao dịch B (rút tiền ra) xảy ra gần như đồng thời trên cùng 1 tài khoản trong giai đoạn dual-write — yêu cầu cả 2 cột `amount_float` và `amount_cents` phải được cập nhật trong cùng transaction với cùng 1 lock trên dòng số dư, không cho phép trạng thái chỉ 1 cột được update.
- Job backfill lịch sử giao dịch phải tính lại `amount_cents` từ `amount_float` gốc (không phải từ số dư đã tính toán ngược), và phải chạy khi hệ thống ở mức traffic thấp theo batch, có cơ chế tạm dừng nếu phát hiện tỷ lệ lệch số dư vượt ngưỡng.
- Có bước validate: sau khi backfill xong mỗi batch, tổng số dư tính theo cột cũ và cột mới của từng user phải khớp trong sai số cho phép (ví dụ dưới 1 cent do rounding), nếu lệch phải chặn không cho tiếp tục sang batch kế và cảnh báo ngay.
- Định nghĩa rollback plan cụ thể: nếu phát hiện lỗi nghiêm trọng sau khi đã chuyển đọc sang cột mới, phải có cách quay lại đọc cột cũ ngay lập tức (feature flag) mà không cần deploy lại code.
