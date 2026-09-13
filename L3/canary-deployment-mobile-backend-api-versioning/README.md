# Canary rollout cho backend mobile phải serving song song nhiều phiên bản app cũ

**Hệ thống:** Một backend phục vụ ứng dụng mobile, trong đó người dùng không update app đồng loạt — tại một thời điểm, backend phải phục vụ đồng thời nhiều phiên bản app khác nhau đang cài trên máy người dùng, có thể chênh nhau nhiều tháng phát hành.

**Vai trò của flow:** Canary deployment ở đây không chỉ kiểm tra phiên bản backend mới có lỗi hay không, mà còn phải đảm bảo phiên bản mới không phá vỡ hợp đồng API mà các phiên bản app cũ đang phụ thuộc, vì không thể ép người dùng update app ngay lập tức.

**Yêu cầu cụ thể:**
- Trước khi canary, phải xác định rõ tập các phiên bản app cũ tối thiểu còn cần hỗ trợ (dựa trên số liệu người dùng thực tế đang dùng phiên bản nào) và test hợp đồng API mới với từng phiên bản đó, không chỉ test với app mới nhất.
- Traffic canary phải được phân chia sao cho bao gồm cả request từ app phiên bản cũ lẫn mới, không chỉ dồn canary vào nhóm user đã update app mới nhất — nếu không, lỗi tương thích ngược chỉ lộ ra sau khi đã mở rộng traffic lên cao.
- Xử lý trường hợp phiên bản backend mới đổi định dạng response (thêm field bắt buộc, đổi kiểu dữ liệu) khiến app cũ không parse được — phải phát hiện qua tỷ lệ crash/lỗi parse tăng bất thường ở nhóm client cũ trong lúc canary, tách biệt với lỗi kỹ thuật chung của backend.
- Đảm bảo cơ chế theo dõi lỗi trong canary phân tách được theo phiên bản app gọi tới (thông qua header version của client), để biết chính xác lỗi phát sinh ở nhóm client nào thay vì gộp chung tỷ lệ lỗi toàn bộ traffic, tránh bỏ sót lỗi chỉ ảnh hưởng một nhóm phiên bản cũ thiểu số.
- Định nghĩa chiến lược rollback không đơn giản là "quay lại code cũ" nếu phiên bản mới đã bắt đầu ghi dữ liệu theo schema mới (ví dụ trường mở rộng cho tính năng mới) — dữ liệu đó vẫn phải đọc được bởi phiên bản cũ đang phục vụ các client legacy sau khi rollback, tránh gãy tương thích ngược thêm lần nữa.
