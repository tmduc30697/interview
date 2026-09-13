# Sequence Diagram - Base: diagnose-slow-checkout

Đây là flow **base** minh hoạ chính vấn đề mà đề bài cần giải quyết: vào giờ cao điểm, checkout bị chậm nhưng đội vận hành không có cách nào xác định chính xác nguyên nhân - có thể do tồn kho bị lock tranh chấp, cổng thanh toán bên ngoài chậm, hay chỉ đơn giản hàng đợi nội bộ nghẽn - vì log của từng service không liên kết với nhau và không tách được thời gian chờ khỏi thời gian xử lý thật. Đây là lý do enhance cần tracing chi tiết theo từng bước.

```mermaid
sequenceDiagram
    actor Ops as Đội vận hành
    participant CartLog as Log cart-service
    participant InvLog as Log inventory-service
    participant PayLog as Log payment-service

    Ops->>CartLog: Xem log giờ cao điểm, thấy nhiều order được tạo
    CartLog-->>Ops: Không thấy tổng thời gian mỗi order xử lý hết bao lâu
    Ops->>InvLog: Xem log inventory-service, đoán có phải do tranh chấp tồn kho
    InvLog-->>Ops: Chỉ thấy "giữ tồn kho thành công", không rõ đã chờ lock bao lâu
    Ops->>PayLog: Xem log payment-service, đoán có phải do cổng thanh toán chậm
    PayLog-->>Ops: Chỉ thấy thời điểm gọi và kết quả, không tách được độ trễ của gateway và của hệ thống
    Note over Ops: Không thể kết luận chính xác nút thắt nằm ở bước nào, phải đoán mò giữa nhiều nguyên nhân khả dĩ
```
