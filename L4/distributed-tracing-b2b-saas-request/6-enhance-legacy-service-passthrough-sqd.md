# Sequence Diagram - Enhance: legacy-service-passthrough

Đây là flow **enhance mới**: `task-service` gọi thêm một service bên thứ ba đã cũ (`legacy-billing-check`) không hỗ trợ tracing, không tạo span nào cả, nhưng vẫn nhận và trả lại nguyên vẹn header `trace_id` (pass-through) để chuỗi trace không bị đứt quãng khi các service tiếp theo tiếp tục xử lý. Đáp ứng yêu cầu 3.

```mermaid
sequenceDiagram
    participant TS as task-service
    participant Legacy as legacy-billing-check (không hỗ trợ tracing)
    participant NS as notification-service

    TS->>TS: Đang xử lý trong span S2 (trace_id=T1, parent_span_id=S1)
    TS->>Legacy: Gọi kiểm tra billing, header: trace_id=T1, parent_span_id=S2
    Note over Legacy: Service legacy không đọc, không tạo span, chỉ xử lý nghiệp vụ
    Legacy-->>TS: Trả kết quả, header trace_id=T1 được giữ nguyên (pass-through)
    TS->>NS: Gọi tiếp notification-service, header: trace_id=T1, parent_span_id=S2
    NS->>NS: Tạo span S3 (trace_id=T1, parent_span_id=S2)
    Note over TS,NS: Chuỗi trace T1 không bị đứt dù đoạn qua legacy-billing-check không có span chi tiết
```
