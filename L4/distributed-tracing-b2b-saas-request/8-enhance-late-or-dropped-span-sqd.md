# Sequence Diagram - Enhance: late-or-dropped-span

Đây là flow **enhance mới**: mỗi span được gửi độc lập tới collector (không phụ thuộc thứ tự nhận), nên khi span của `search-indexer` cho trace T1 bị gửi trễ do network drop, collector vẫn dựng đúng cây trace của T1 ngay khi span tới muộn, và trace T2 của 1 request khác xử lý song song hoàn toàn không bị ảnh hưởng. Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    participant SI as search-indexer (trace T1)
    participant NS as notification-service (trace T1)
    participant OtherSvc as task-service (trace T2, request khác)
    participant Collector as Tracing Collector/Backend

    NS->>Collector: Gửi span S3 (trace_id=T1, parent_span_id=S2)
    Collector->>Collector: Lưu span S3, gắn vào cây trace T1
    OtherSvc->>Collector: Gửi span của trace T2 (không liên quan T1)
    Collector->>Collector: Lưu span T2 độc lập, không ảnh hưởng tới T1
    Note over SI,Collector: Span S4 của search-indexer (trace T1) bị mất gói trên đường truyền, chưa tới collector
    Collector->>Collector: Hiển thị cây trace T1 tạm thời thiếu S4, các span khác vẫn đúng
    SI->>Collector: Gửi lại/gửi trễ span S4 (trace_id=T1, parent_span_id=S2)
    Collector->>Collector: Nhận span S4 trễ, tự bổ sung đúng vị trí vào cây trace T1 nhờ span tự chứa đủ thông tin
    Note over Collector: Trace T1 được dựng lại đầy đủ, trace T2 không bị sai lệch trong suốt quá trình
```
