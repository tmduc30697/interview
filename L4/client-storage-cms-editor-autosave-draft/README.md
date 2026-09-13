# Autosave draft chống mất nội dung bằng IndexedDB cho trình soạn thảo CMS

**Hệ thống:** Trình soạn thảo nội dung dạng rich-text (CMS viết bài, hoặc công cụ soạn tài liệu) cho phép soạn bài dài, cần autosave draft liên tục vào IndexedDB để không mất nội dung khi crash trình duyệt/mất điện, trước khi kịp lưu lên server.

**Vai trò của flow:** Đảm bảo "không bao giờ mất nội dung đang gõ" mà không làm chậm trải nghiệm gõ phím, và không tạo xung đột khi cùng lúc có bản draft cục bộ và bản đã lưu trên server.

**Yêu cầu cụ thể:**
- Autosave vào IndexedDB theo debounce (ví dụ mỗi 2-3 giây sau khi ngừng gõ) thay vì lưu mỗi keystroke, để không chặn main thread khi nội dung dài (hàng chục nghìn từ) — đo rõ ngưỡng độ dài nội dung mà việc serialize/ghi IndexedDB bắt đầu gây giật khi gõ.
- Khi mở lại app sau khi crash/đóng nhầm tab, phải phát hiện có draft chưa lưu trong IndexedDB mới hơn bản đã lưu trên server, và hỏi rõ user chọn khôi phục draft cục bộ hay giữ bản trên server — không tự động ghi đè một trong hai một cách ngầm định.
- Mở cùng 1 bài viết ở 2 tab/2 máy khác nhau: draft cục bộ trong IndexedDB chỉ có ý nghĩa cục bộ (không đồng bộ qua lại giữa các máy), nên khi server đã có bản mới hơn (do sửa từ máy/tab khác) phải cảnh báo rõ nguy cơ mất nội dung nếu cứ lưu đè, thay vì im lặng ghi đè draft cũ hơn.
- Dọn dẹp draft trong IndexedDB sau khi đã lưu thành công lên server hoặc bài viết đã publish/xóa, tránh tồn đọng draft rác vô thời hạn làm phình dung lượng IndexedDB và có thể vô tình khôi phục nhầm draft đã lỗi thời cho một bài viết đã bị xóa từ lâu.
- Xử lý khi IndexedDB bị lỗi/không khả dụng (một số trình duyệt chặn trong chế độ ẩn danh hoặc storage bị corrupt) — có fallback về autosave lên server với tần suất thấp hơn, và cảnh báo rõ cho user biết chế độ "an toàn khi crash cục bộ" đang không hoạt động.
