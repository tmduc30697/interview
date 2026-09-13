# Chỉnh sửa cấu hình hệ thống nội bộ (feature flag, config) bởi nhiều kỹ sư

**Hệ thống:** Một dashboard nội bộ cho kỹ sư quản lý feature flag/config runtime của hệ thống production, thay đổi có hiệu lực ngay khi lưu, nhiều kỹ sư ở các team khác nhau có quyền chỉnh sửa.

**Vai trò của flow:** Vì thay đổi config có tác động ngay tới production và tần suất nhiều kỹ sư cùng sửa 1 config trong cùng thời điểm là thấp nhưng hậu quả nếu ghi đè nhầm khá nghiêm trọng, flow nên dùng optimistic locking kết hợp với hiển thị rõ diff thay đổi trước khi cho phép lưu đè.

**Yêu cầu cụ thể:**
- Mỗi config có version, khi kỹ sư A lưu thay đổi dựa trên version cũ mà đã có kỹ sư B lưu thay đổi khác trước đó (version mới hơn), hệ thống phải từ chối lưu và hiển thị rõ diff giữa bản A đang muốn lưu, bản gốc A đã đọc, và bản mới nhất B đã lưu — để A tự quyết định merge tay chính xác, không tự động merge ngầm với config nhạy cảm ảnh hưởng production.
- Mô tả cụ thể: kỹ sư A đang tắt feature flag X (do phát hiện lỗi khẩn cấp), đúng lúc kỹ sư B đang sửa 1 config khác không liên quan trong cùng bộ config đó nhưng version chung bị tăng do B lưu trước — cần đánh giá xem có nên tách optimistic lock ở cấp độ từng flag/config riêng lẻ (để A không bị chặn bởi thay đổi không liên quan của B) thay vì version chung cho cả nhóm config, đặc biệt quan trọng cho action khẩn cấp như tắt flag khi có sự cố.
- Với action khẩn cấp (ví dụ "tắt flag ngay" trong sự cố production), nên có đường xử lý riêng ưu tiên ghi đè ngay không chờ optimistic lock thông thường (bỏ qua version check hoặc có nút "buộc lưu" rõ ràng có cảnh báo), vì trong tình huống khẩn cấp việc bị chặn bởi conflict version có thể gây chậm trễ nguy hiểm hơn rủi ro ghi đè nhầm — nhưng phải log rõ ràng và cảnh báo action này đã bỏ qua kiểm tra xung đột.
- Mọi thay đổi config (dù qua optimistic lock thông thường hay đường khẩn cấp) phải được ghi lịch sử đầy đủ và không thể xóa (audit log immutable): ai, khi nào, giá trị trước/sau, có phải action khẩn cấp không — để phục vụ điều tra sau sự cố nếu 1 thay đổi config gây ảnh hưởng không mong muốn.
- Có cơ chế thông báo real-time cho các kỹ sư khác đang mở màn hình chỉnh sửa cùng 1 config khi có ai đó vừa lưu thay đổi (qua WebSocket/polling), để giảm khả năng họ submit dựa trên version đã lỗi thời và phải xử lý conflict muộn hơn cần thiết.
