# Sequence Diagram - Enhance: payment-external-dependency

Đây là flow **enhance mới**: span của bước gọi cổng thanh toán được đánh dấu `is_external_dependency=true` kèm `external_sla_ms` riêng, để khi tổng hợp thống kê "checkout chậm ở bước nào" vào giờ cao điểm, đội vận hành phân biệt được độ trễ do cổng thanh toán đối tác (nằm ngoài tầm kiểm soát, so với SLA riêng của họ) và độ trễ do chính hệ thống nội bộ gây ra. Đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant Pay as payment-service
    participant Gateway as Cổng thanh toán bên ngoài (SLA cam kết 500ms)
    participant Tracer as Tracing SDK

    Pay->>Tracer: Bắt đầu span S3, gắn tag is_external_dependency=true, external_sla_ms=500
    Pay->>Gateway: Gọi xử lý thanh toán
    Note over Gateway: Cổng thanh toán xử lý mất 1200ms, vượt SLA cam kết 500ms
    Gateway-->>Pay: Trả kết quả thanh toán thành công
    Pay->>Tracer: Kết thúc span S3, duration=1200ms
    Tracer->>Tracer: Ghi nhận span external, so sánh duration với external_sla_ms
    Note over Tracer: Khi tổng hợp giờ cao điểm, độ trễ vượt SLA của span external này được quy đúng cho cổng thanh toán, không tính là lỗi hệ thống nội bộ
```
