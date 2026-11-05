# Blue-green deployment cho service lõi của nền tảng SaaS multi-tenant

**Hệ thống:** Một nền tảng SaaS B2B phục vụ nhiều tenant (công ty khách hàng) dùng chung hạ tầng, service "core-api" được deploy nhiều lần mỗi tuần.

**Vai trò của flow:** Blue-green deployment cho phép chuyển toàn bộ traffic từ phiên bản cũ (blue) sang phiên bản mới (green) gần như ngay lập tức, và rollback tức thì nếu phát hiện sự cố sau khi chuyển.

**Yêu cầu cụ thể:**
- Trước khi chuyển traffic, phiên bản green phải được deploy đầy đủ song song với blue (không tắt blue), và trải qua bước smoke test tự động (gọi các endpoint quan trọng) xác nhận hoạt động đúng trước khi router bắt đầu chuyển traffic thật sang.
- Việc chuyển traffic từ blue sang green phải là một thao tác gần như tức thời ở tầng router/load balancer (đổi target), không yêu cầu client phải làm gì, và phải có khả năng đảo ngược (chuyển lại về blue) trong vài giây nếu phát hiện lỗi ngay sau khi chuyển.
- Xử lý đúng vấn đề tương thích dữ liệu khi có migration schema DB đi kèm bản deploy mới — phiên bản blue (cũ) vẫn đang chạy phải tiếp tục hoạt động đúng với schema đã thay đổi cho tới khi bị tắt hẳn (yêu cầu migration phải backward-compatible trong giai đoạn chuyển tiếp).
- Định nghĩa rõ tiêu chí tự động rollback (ví dụ tỷ lệ lỗi 5xx vượt ngưỡng X% trong Y phút đầu sau khi chuyển) và cơ chế giám sát phải chạy ngay sau khi chuyển traffic, không chờ phát hiện thủ công từ báo cáo người dùng.
- Đảm bảo các session/kết nối đang mở tại thời điểm chuyển đổi (ví dụ WebSocket, request dài) được xử lý đúng — không bị cắt ngang giữa lúc router đổi target, mà phải hoàn tất trên blue hoặc được chuyển tiếp một cách có kiểm soát.
