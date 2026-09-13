# Fintech tra cứu lịch sử giao dịch trên bảng transaction khổng lồ

**Hệ thống:** Ngân hàng số/ví điện tử có bảng `transactions` append-only với hàng trăm triệu dòng, phục vụ cả user tra cứu lịch sử giao dịch của mình lẫn CSKH/vận hành tra cứu theo nhiều tiêu chí khác nhau.

**Vai trò của flow:** Đảm bảo tra cứu nhanh và chính xác trong khi bảng vẫn đang bị ghi liên tục với tốc độ cao (giao dịch mới insert từng giây), mà không phá vỡ yêu cầu audit trail toàn vẹn.

**Yêu cầu cụ thể:**
- User tra lịch sử giao dịch của tài khoản mình cần composite index (account_id, created_at DESC) để lấy nhanh giao dịch gần nhất, nhưng CSKH lại cần lọc thêm theo status hoặc transaction_type trên cùng bảng — quyết định rõ có thêm cột vào index hiện tại, tạo index riêng, hay chấp nhận index intersection cho các trường hợp ít gặp hơn.
- Bảng append-only tăng kích thước liên tục khiến index B-tree ngày càng phình to, làm chậm dần cả tốc độ ghi lẫn thời gian bảo trì index — cần cân nhắc range-partition bảng theo tháng/năm (dựa trên created_at) kết hợp local index riêng từng partition để giữ mỗi index luôn nhỏ gọn.
- Truy vấn "toàn bộ giao dịch failed/pending" chỉ chiếm tỉ lệ rất nhỏ trên tổng bảng nhưng được CSKH chạy thường xuyên để xử lý sự cố — index toàn bảng theo status sẽ lãng phí vì đa số giao dịch đã success; cần dùng partial index (chỉ index các dòng có status IN pending/failed) để index nhỏ và nhanh hơn hẳn.
- Bộ phận compliance cần chạy query audit full-scan theo nhiều tổ hợp cột không thể index hết (đổi tiêu chí liên tục theo từng đợt kiểm toán) — cần quyết định có tách hẳn read replica riêng cho audit/reporting để không cạnh tranh tài nguyên index/buffer pool với luồng giao dịch thời gian thực.
- Giờ cao điểm có rất nhiều giao dịch insert đồng thời, index có cột đơn điệu tăng dần (như created_at hoặc id tự tăng) dễ tạo hot page ở cuối B-tree gây lock contention khi nhiều transaction cùng ghi — cần xem xét chiến lược giảm contention (vd tách theo shard/account_id trước khi index theo thời gian).
