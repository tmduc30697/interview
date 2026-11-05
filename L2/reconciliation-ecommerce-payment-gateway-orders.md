# E-commerce đối soát giao dịch giữa cổng thanh toán và hệ thống đơn hàng

**Hệ thống:** Một sàn e-commerce nhận thanh toán qua cổng thanh toán bên thứ ba (ví dụ Stripe/VNPay) cho các đơn hàng.

**Vai trò của flow:** Đối soát định kỳ giữa danh sách giao dịch từ cổng thanh toán và trạng thái đơn hàng trong hệ thống nội bộ để phát hiện sai lệch (đơn báo đã thanh toán nhưng cổng không ghi nhận, hoặc ngược lại).

**Yêu cầu cụ thể:**
- Mỗi giao dịch từ cổng thanh toán phải được khớp (match) với đúng một đơn hàng nội bộ dựa trên mã tham chiếu duy nhất, phát hiện được các trường hợp không khớp (giao dịch không có đơn tương ứng, hoặc đơn báo đã trả tiền nhưng không tìm thấy giao dịch).
- Xử lý đúng các giao dịch bị hoàn tiền (refund) hoặc bị đảo (chargeback) xảy ra sau khi đơn hàng đã đối soát thành công trước đó — phải cập nhật lại trạng thái đối soát tương ứng, không để đơn hàng hiển thị "đã đối soát khớp" khi thực tế tiền đã được hoàn.
- Báo cáo đối soát phải phân loại rõ các loại sai lệch (thiếu giao dịch, thừa giao dịch, sai số tiền, sai trạng thái) để đội vận hành xử lý đúng quy trình cho từng loại, không gộp chung một danh sách "lỗi".
- Job đối soát phải chạy được lại nhiều lần trên cùng một khoảng thời gian dữ liệu mà không tạo ra báo cáo trùng lặp hoặc đánh dấu sai lệch nhiều lần cho cùng một giao dịch đã được xử lý.
- Các sai lệch không tự giải quyết được (cần con người can thiệp) phải được đưa vào hàng đợi xử lý thủ công có theo dõi trạng thái, đảm bảo không có sai lệch nào bị bỏ quên không xử lý.
