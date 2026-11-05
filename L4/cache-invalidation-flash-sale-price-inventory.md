# Cache giá/tồn kho trong flash sale với tần suất thay đổi cực cao

**Hệ thống:** E-commerce trong giờ flash sale có giá và tồn kho thay đổi liên tục (giảm giá theo giờ, tồn kho giảm nhanh), cache phải theo kịp thay đổi mà vẫn chịu được tải đọc khổng lồ.

**Vai trò của flow:** Invalidate cache tồn kho/giá gần như real-time trong khi vẫn giữ được throughput đọc cao, khác với cache thông thường có thể chấp nhận độ trễ lớn hơn.

**Yêu cầu cụ thể:**
- Cache tồn kho phải có TTL rất ngắn (giây) kết hợp active invalidation khi có đơn hàng trừ kho, để giảm thiểu rủi ro hiển thị "còn hàng" sai trong lúc sản phẩm sắp hết.
- Với sản phẩm sắp hết hàng (dưới ngưỡng an toàn), phải chuyển sang đọc trực tiếp từ nguồn (bỏ qua cache) để đảm bảo chính xác, đánh đổi tăng tải DB cho nhóm sản phẩm nhỏ này để bù lại độ chính xác.
- Xử lý cache stampede khi giá flash sale bắt đầu/kết thúc đúng giờ (hàng loạt cache cùng invalidate đồng thời tại thời điểm giờ chốt) — cần pre-warm cache trước thời điểm đổi giá vài giây để tránh dội tải.
- Đảm bảo giá hiển thị cho user tại thời điểm checkout khớp với giá thực tế lúc submit đơn (double-check giá ở backend, không tin hoàn toàn giá đã cache hiển thị ở client).
- Đo lường: tỉ lệ hiển thị sai giá/tồn kho (nếu có) trong log thực tế của các lần flash sale trước, dùng làm baseline để cải thiện chiến lược invalidation.
