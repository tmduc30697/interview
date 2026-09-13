# Logistics tra cứu và theo dõi vận đơn với tần suất ghi trạng thái cao

**Hệ thống:** Nền tảng logistics quản lý vận đơn (shipment), mỗi vận đơn được cập nhật trạng thái liên tục nhiều lần trong ngày (pickup, in-transit, out-for-delivery, delivered...) qua nhiều trạm trung chuyển khác nhau.

**Vai trò của flow:** Cho phép khách hàng và bộ phận vận hành tra cứu nhanh theo mã vận đơn, theo trạng thái hiện tại, hoặc theo khoảng thời gian, trong khi bảng dữ liệu liên tục bị ghi cập nhật với tần suất cao.

**Yêu cầu cụ thể:**
- Unique index trên `tracking_code` phải đảm bảo khách hàng tra cứu tức thời, nhưng nếu mã vận đơn được sinh ngẫu nhiên không tuần tự, việc insert liên tục vận đơn mới có thể gây phân mảnh index nhanh hơn — cần cân nhắc cách sinh mã vừa đảm bảo tính duy nhất/khó đoán, vừa thân thiện với cấu trúc index.
- Dashboard vận hành cần lọc vận đơn theo trạng thái hiện tại, nhưng trạng thái đổi liên tục nghĩa là mỗi lần update phải xoá và chèn lại entry trong index theo status — với hàng triệu vận đơn đang active, chi phí ghi này rất lớn; cần cân nhắc partial index chỉ bao phủ các trạng thái "đang xử lý" thay vì index toàn bộ trạng thái kể cả các đơn đã delivered từ lâu.
- Lịch sử thay đổi trạng thái (nhiều dòng cho mỗi vận đơn) cần composite index (shipment_id, changed_at) để trả đúng thứ tự timeline cho khách xem, nhưng bảng lịch sử tăng vô hạn theo thời gian — cần có chiến lược archive/partition dữ liệu cũ mà không phá vỡ hiệu năng của index đang phục vụ dữ liệu gần đây.
- Hệ thống cần chạy định kỳ (vd mỗi phút) một truy vấn tìm các vận đơn trễ SLA (estimated_delivery đã qua nhưng status khác delivered) trên bảng có quy mô lớn — quét toàn bảng mỗi phút là không khả thi, cần thiết kế index (partial/composite) riêng tối ưu cho đúng dạng truy vấn cảnh báo này.
- Nhiều trạm trung chuyển ghi cập nhật trạng thái đồng thời cho các vận đơn khác nhau — cần chọn cấu trúc index/storage tránh việc nhiều ghi đồng thời dồn vào cùng vài trang index "nóng" (hot page), gây lock contention làm chậm toàn bộ hệ thống dù các vận đơn được cập nhật không liên quan tới nhau.
