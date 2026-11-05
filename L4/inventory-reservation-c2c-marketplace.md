# Giữ tồn kho khi seller và buyer thao tác đồng thời trên marketplace C2C

**Hệ thống:** Một marketplace C2C (kiểu Facebook Marketplace) cho phép seller đăng nhiều sản phẩm với số lượng, buyer có thể đặt mua và seller cũng có thể tự sửa/xóa số lượng bất cứ lúc nào.

**Vai trò của flow:** Flow reservation phải xử lý được xung đột giữa hành động buyer (đặt mua, giữ tồn kho) và hành động seller (tự sửa số lượng, ngừng bán) xảy ra đồng thời trên cùng 1 sản phẩm.

**Yêu cầu cụ thể:**
- Khi seller sửa số lượng tồn kho từ 5 xuống 2 đúng lúc có 3 buyer khác nhau đang giữ reservation (tổng 3 đơn vị đã reserve), quy định rõ hành vi: các reservation đã tạo trước khi seller sửa vẫn hợp lệ (không tự hủy), chỉ số lượng còn lại để bán mới bị giới hạn theo giá trị mới — không để số liệu "âm" (reserved vượt tồn kho mới) gây lỗi hiển thị.
- Khi seller bấm "Ngừng bán" (đóng sản phẩm) đúng lúc buyer đang ở giữa bước xác nhận đặt mua đã giữ reservation trước đó, request xác nhận của buyer phải được xử lý dựa trên trạng thái tại thời điểm reservation được tạo (buyer đã giữ hợp lệ thì vẫn được mua), còn buyer mới sau khi sản phẩm đã đóng không thể tạo reservation mới — mô tả rõ ranh giới thời điểm nào được tính là "trước/sau".
- 2 buyer cùng bấm "Mua ngay" cho sản phẩm chỉ còn 1 số lượng gần như đồng thời: chỉ 1 người giữ được reservation, thiết kế bằng update nguyên tử trên cột tồn kho khả dụng, có test giả lập concurrency xác nhận không có 2 reservation cùng tồn tại cho 1 đơn vị hàng.
- Reservation của buyer phải có TTL ngắn hơn ở marketplace C2C so với e-commerce chính thức (vì tốc độ trao đổi/chat trước khi chốt mua có thể chậm), quy định giá trị TTL và cách gia hạn khi buyer và seller đang chat để thống nhất giao dịch.
- Có audit log cho mọi thay đổi tồn kho (ai sửa, khi nào, giá trị trước/sau) để xử lý tranh chấp khi buyer khiếu nại "đã đặt được hàng nhưng sau đó bị báo hết".
