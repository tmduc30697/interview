# Sequence Diagram - Enhance: inventory-lock-wait-vs-processing

Đây là flow **enhance mới**: hai order cùng tranh chấp giữ tồn kho cho cùng 1 sản phẩm hot, span của `inventory-service` tách rõ `wait_time_ms` (thời gian order 2 phải chờ lock được order 1 nhả) khỏi `processing_time_ms` (thời gian xử lý logic giữ tồn kho thật sự), giúp phân biệt "chậm vì tranh chấp tồn kho" với "chậm vì code xử lý nặng". Đáp ứng yêu cầu 2.

```mermaid
sequenceDiagram
    participant Cart1 as cart-service (order 1)
    participant Cart2 as cart-service (order 2)
    participant Inv as inventory-service
    participant Lock as Lock sản phẩm hot (SKU-X)

    Cart1->>Inv: Yêu cầu giữ tồn kho SKU-X, trace_id=TR1
    Inv->>Lock: Acquire lock SKU-X
    Lock-->>Inv: Cấp lock ngay (không ai giữ)
    Inv->>Inv: Xử lý giữ tồn kho thật (processing_time_ms=20ms)
    Cart2->>Inv: Yêu cầu giữ tồn kho SKU-X, trace_id=TR2 (gần như cùng lúc)
    Inv->>Lock: Acquire lock SKU-X cho order 2
    Lock-->>Inv: Phải chờ, order 1 đang giữ lock
    Inv->>Inv: Span order 2 bắt đầu đếm wait_time_ms
    Inv-->>Cart1: Giữ tồn kho order 1 thành công, release lock
    Lock-->>Inv: Cấp lock cho order 2 (đã chờ wait_time_ms=180ms)
    Inv->>Inv: Kết thúc đếm chờ, bắt đầu processing_time_ms=20ms cho order 2
    Inv-->>Cart2: Giữ tồn kho order 2 thành công
    Note over Inv: Span order 2 ghi wait_time_ms=180ms, processing_time_ms=20ms riêng biệt, không gộp chung vào 1 con số
```
