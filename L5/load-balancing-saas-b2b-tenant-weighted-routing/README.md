# API gateway route theo tenant có trọng số cho nền tảng SaaS B2B

**Hệ thống:** Một API gateway đứng trước các service backend của nền tảng SaaS B2B, phục vụ nhiều tenant (công ty khách hàng) có quy mô rất khác nhau — một số tenant lớn gửi lượng request gấp hàng trăm lần tenant nhỏ.

**Vai trò của flow:** Load balancing ở đây không chỉ phân phối request đều tới các instance backend, mà còn phải đảm bảo công bằng tài nguyên giữa các tenant — một tenant lớn đột biến traffic không được phép chiếm hết capacity chung khiến các tenant nhỏ hơn bị chậm hoặc timeout.

**Yêu cầu cụ thể:**
- Thiết kế cơ chế giới hạn tài nguyên theo tenant (ví dụ số connection đồng thời hoặc số request/giây tối đa mà một tenant được phép chiếm trên tổng capacity cụm) độc lập với thuật toán chọn instance, để một tenant có traffic bất thường bị giới hạn ở tầng gateway trước khi kịp làm nghẽn backend chung.
- Xử lý trường hợp một tenant lớn đang có traffic đột biến hợp lệ (không phải tấn công, ví dụ họ đang chạy chiến dịch của riêng họ) — phân biệt được với traffic bất thường/lỗi client để không chặn nhầm nhu cầu chính đáng, đồng thời vẫn bảo vệ được tenant khác.
- Đảm bảo việc đếm và giới hạn traffic theo tenant hoạt động đúng khi có nhiều instance gateway chạy song song (bộ đếm phải chia sẻ trạng thái giữa các instance gateway, không phải đếm riêng độc lập từng instance rồi mỗi instance tự cho qua một phần giới hạn dẫn đến tổng vượt ngưỡng thực tế).
- Cho phép cấu hình trọng số route khác nhau theo gói dịch vụ của tenant (tenant trả phí cao hơn được ưu tiên nhiều tài nguyên hơn khi cụm gần đạt tải tối đa) mà không cần phải tách riêng instance backend vật lý cho từng nhóm tenant, tránh lãng phí tài nguyên khi tenant lớn đang rảnh.
- Thiết kế test mô phỏng một tenant gửi traffic tăng đột biến 50 lần trong vài giây, đo độ trễ và tỷ lệ lỗi của các tenant khác trong cùng cụm — phải chứng minh được các tenant khác gần như không bị ảnh hưởng nhờ giới hạn tài nguyên theo tenant đã thiết kế.
