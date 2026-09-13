# Enhance sequence — In-flight order rollback handling

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm "đơn dở dang trên 1 version cụ thể"). Đáp ứng yêu cầu thứ 3 của đề bài: đơn hàng đã bắt đầu ở canary (đã giữ tồn kho, đã tạo payment intent) nhưng canary bị rollback giữa chừng phải được hoàn tất hoặc hoàn tác nhất quán, không để treo dở dang.

```mermaid
sequenceDiagram
    participant Router as Router (vừa rollback)
    participant Canary as Deployment canary (vẫn đang chạy, chưa bị tắt)
    participant DB as ORDER store
    participant Gateway as Payment Gateway
    participant Inv as Inventory
    participant Resolution as IN_FLIGHT_ORDER_RESOLUTION store

    Note over Router: ROLLBACK_EVENT vừa xảy ra, canary_percent=0, nhưng Canary chưa bị tắt hẳn ngay
    Router->>DB: Tìm toàn bộ ORDER có handled_by_deployment_id=canary và status=pending
    loop Với mỗi đơn dở dang trên canary
        DB->>Canary: Kiểm tra trạng thái xử lý hiện tại của đơn này
        alt Canary vẫn có thể hoàn tất an toàn (đã gọi gateway, đang chờ phản hồi hợp lệ)
            Canary->>Gateway: Cho phép hoàn tất nốt lời gọi đang dở
            Gateway-->>Canary: Kết quả thanh toán
            Canary->>DB: Cập nhật ORDER.status theo kết quả
            Canary->>Resolution: Ghi resolution=completed_on_canary
        else Trạng thái không an toàn để tiếp tục trên canary (vd lỗi/timeout đang xảy ra)
            Canary->>Gateway: Void/huỷ payment intent dở dang (nếu có)
            Canary->>Inv: Nhả lại tồn kho đã giữ
            DB->>DB: Đặt ORDER.status=retry_pending
            Router->>DB: Đơn được retry lại từ đầu trên Stable
            DB->>Resolution: Ghi resolution=compensated_and_retried_on_stable
        end
    end
    Note over DB,Resolution: Không có đơn nào bị bỏ lại ở trạng thái pending vô thời hạn sau khi traffic đã chuyển hết về stable
```
