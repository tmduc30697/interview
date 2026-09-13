# Ngân hàng số đối soát giao dịch với đối tác thanh toán/mạng thẻ

**Hệ thống:** Một ngân hàng số (neobank) xử lý giao dịch thẻ và chuyển tiền qua các đối tác bên ngoài (mạng thẻ, cổng chuyển mạch liên ngân hàng).

**Vai trò của flow:** Đối soát cuối ngày giữa sổ cái (ledger) nội bộ của ngân hàng và file/báo cáo giao dịch nhận từ đối tác thanh toán để đảm bảo mọi giao dịch được ghi nhận đúng và số dư khách hàng chính xác.

**Yêu cầu cụ thể:**
- Đối soát phải xử lý được file batch từ đối tác thanh toán có thể đến muộn hoặc theo múi giờ khác với hệ thống nội bộ — phải định nghĩa rõ "ngày giao dịch" theo cùng một chuẩn giữa hai bên để tránh lệch ngày khi đối soát.
- Phát hiện chính xác các loại sai lệch khác nhau: giao dịch có ở đối tác nhưng thiếu trong ledger nội bộ, giao dịch có trong ledger nhưng đối tác không xác nhận, và giao dịch khớp nhưng sai số tiền — mỗi loại phải có quy trình xử lý và mức độ ưu tiên khác nhau vì đây là tiền thật của khách hàng.
- Giao dịch bị đối tác báo hoàn/đảo sau một khoảng thời gian (ví dụ tranh chấp thẻ) phải được xử lý đúng dù giao dịch gốc đã đối soát khớp từ trước, và số dư khách hàng phải được điều chỉnh đúng thời điểm phát hiện.
- Toàn bộ quy trình đối soát và mọi điều chỉnh số dư phát sinh từ đó phải có audit trail đầy đủ, không thể sửa trực tiếp số dư mà không qua bước ghi nhận giao dịch điều chỉnh tương ứng, để đáp ứng yêu cầu kiểm toán ngân hàng.
- Phải có ngưỡng cảnh báo tự động khi tổng sai lệch chưa xử lý trong ngày vượt quá một số tiền nhất định, yêu cầu can thiệp khẩn của đội vận hành trước khi đóng sổ ngày đó.
