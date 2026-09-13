# Nâng/hạ cấp gói subscription xuyên nhiều service (billing + entitlement + notification)

**Hệ thống:** SaaS B2B cho phép khách hàng nâng cấp/hạ cấp plan, việc này phải đồng bộ giữa billing (tính tiền), entitlement (mở/khóa tính năng), và notification (báo khách).

**Vai trò của flow:** Saga đảm bảo billing và entitlement luôn khớp nhau — không có trường hợp khách bị charge tiền plan mới nhưng chưa được mở tính năng, hoặc ngược lại.

**Yêu cầu cụ thể:**
- Thứ tự saga: charge/pro-rate billing trước, chỉ mở entitlement sau khi billing xác nhận thành công; nếu billing thất bại, không chạm tới entitlement.
- Nếu entitlement service down đúng lúc cần cập nhật sau khi billing đã thành công, saga phải retry với backoff trong một khoảng thời gian giới hạn, và nếu vẫn thất bại phải compensate (hoàn tiền phần chênh lệch) và giữ plan cũ.
- Downgrade plan phải xử lý được trường hợp khách đang dùng tính năng vượt hạn mức plan mới (ví dụ vượt số user) — quyết định rõ: chặn downgrade hay downgrade kèm cảnh báo.
- Notification chỉ được gửi sau khi cả billing và entitlement đã ổn định (không gửi email "đã nâng cấp thành công" rồi sau đó saga phải rollback).
- Toàn bộ lịch sử thay đổi plan của một khách phải truy vấn lại được (ai đổi plan gì, khi nào, kết quả ra sao) để phục vụ hỗ trợ khách hàng.
