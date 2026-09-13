# Sequence Diagram - Enhance: payment-timeout-late-callback-reconciliation

Đây là flow **enhance mới**: order bị timeout ở bước chờ callback thanh toán và tạm thời được đánh dấu thất bại, nhưng cổng thanh toán thực ra đã xử lý thành công phía họ và gửi callback trễ sau đó. Nhờ span thanh toán và `PAYMENT_CALLBACK_EVENT` đều lưu đủ `trace_id`/`span_id`, hệ thống đối chiếu được callback trễ với đúng order để sửa lại trạng thái, tránh báo "thất bại" trong khi tiền đã bị trừ. Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    participant Cart as cart-service
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài
    participant Confirm as order-confirmation-service
    actor Customer

    Cart->>Pay: Yêu cầu thanh toán, trace_id=TR7, tạo span S3
    Pay->>Gateway: Gọi xử lý thanh toán
    Note over Pay: Chờ callback quá timeout quy định (ví dụ 10s) mà chưa nhận được phản hồi
    Pay->>Pay: Đánh dấu span S3 status=TIMEOUT, order tạm thời status=FAILED
    Pay-->>Cart: Báo thanh toán thất bại (tạm thời)
    Cart-->>Customer: Thông báo đơn hàng thất bại
    Note over Gateway: Thực ra Gateway đã xử lý thành công ngay sau đó, chỉ gửi callback bị trễ
    Gateway-->>Pay: Callback trễ báo thanh toán thành công, kèm trace_id=TR7, gateway_reference=GW-889
    Pay->>Pay: Ghi PAYMENT_CALLBACK_EVENT, liên kết ngược về span S3 qua trace_id=TR7
    Pay->>Pay: Đối chiếu, phát hiện callback trễ khớp với order đã báo FAILED, đánh dấu matched_after_timeout=true
    Pay->>Confirm: Sửa lại trạng thái order thành đã thanh toán thành công
    Confirm-->>Customer: Cập nhật lại đơn hàng đã thanh toán thành công, không cần khiếu nại
```
