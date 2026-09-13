# Checkout e-commerce nhiều bước (payment + inventory + shipping)

**Hệ thống:** Trang e-commerce với luồng đặt hàng chạm tới 3 service riêng: Payment, Inventory, Shipping.

**Vai trò của flow:** Saga orchestration điều phối toàn bộ transaction nghiệp vụ xuyên service, và compensate (hủy ngược) khi một bước giữa đường thất bại.

**Yêu cầu cụ thể:**
- Định nghĩa rõ từng bước saga (reserve inventory → charge payment → create shipment) và compensating action tương ứng cho mỗi bước (release inventory, refund payment, cancel shipment).
- Nếu bước charge payment thất bại sau khi đã reserve inventory thành công, phải tự động gọi compensate để release inventory, và toàn bộ order chuyển trạng thái "failed" chứ không kẹt ở trạng thái lửng lơ.
- Saga state (đang ở bước nào, đã compensate bước nào) phải được persist, để nếu orchestrator service crash giữa saga, có thể resume đúng chỗ khi restart.
- Xử lý được trường hợp compensate cũng thất bại (ví dụ refund payment lỗi) — phải có retry với backoff và escalate sang xử lý thủ công/alert nếu vượt số lần retry.
- Toàn bộ event của saga (bắt đầu, mỗi step, compensate) phải được lưu dưới dạng event log để có thể replay/audit lại toàn bộ hành trình của một order.
