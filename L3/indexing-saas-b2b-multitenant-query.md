# SaaS B2B tối ưu index cho truy vấn đa tenant dùng chung bảng

**Hệ thống:** SaaS B2B multi-tenant dùng shared schema — nhiều công ty khách hàng (tenant) lưu dữ liệu chung trong cùng một bộ bảng, phân biệt bằng cột `tenant_id`, với quy mô dữ liệu lệch rất lớn giữa các tenant (tenant lớn hàng triệu dòng, tenant nhỏ chỉ vài trăm dòng).

**Vai trò của flow:** Đảm bảo mỗi tenant tra cứu dữ liệu của riêng mình luôn nhanh và cách ly đúng, bất kể tenant khác đang có khối lượng dữ liệu hoặc truy vấn lớn tới đâu (tránh noisy-neighbor ảnh hưởng lẫn nhau qua tầng index/cache dùng chung).

**Yêu cầu cụ thể:**
- Mọi composite index trên các bảng dùng chung đều phải đặt `tenant_id` làm cột đầu tiên để mỗi truy vấn luôn lọc được theo tenant trước — nếu một tính năng mới quên tuân thủ quy tắc này, query sẽ rơi vào full table scan chậm dần và có nguy cơ vô tình đọc chéo dữ liệu tenant khác do lỗi logic đi kèm.
- Do độ lệch dữ liệu lớn giữa các tenant, query planner có thể chọn plan khác nhau cho cùng một câu query tùy vào cardinality của tenant đang chạy (tenant nhỏ có thể bị ép dùng index kém hiệu quả, tenant lớn có thể bị planner chọn nhầm sang full scan) — cần theo dõi thống kê (ANALYZE) và xử lý rõ trường hợp planner chọn sai plan.
- Một số tenant cần lưu field tùy biến riêng (custom fields) qua cột JSONB dùng chung cho mọi tenant — B-tree thường không index hiệu quả cho các field động này, cần đánh giá dùng GIN index hoặc expression index cho từng loại field hay truy vấn phổ biến.
- Khi một tenant lớn chạy report/aggregation nặng, nó có thể chiếm phần lớn buffer pool/index cache dùng chung, khiến các tenant nhỏ khác bị chậm theo dù không liên quan tới truy vấn đó — cần cân nhắc tách index/replica riêng cho luồng reporting nặng khỏi luồng giao dịch thời gian thực.
- Thêm index mới cho một tính năng phải chạy migration online (kiểu CREATE INDEX CONCURRENTLY) trên bảng chứa dữ liệu của toàn bộ tenant cùng lúc — cần đảm bảo quá trình build index không khoá ghi và không làm gián đoạn tenant nào, kể cả tenant đang có traffic cao nhất tại thời điểm migrate.
