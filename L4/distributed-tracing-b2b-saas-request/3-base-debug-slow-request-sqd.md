# Sequence Diagram - Base: debug-slow-request

Đây là flow **base** minh hoạ chính vấn đề mà đề bài cần giải quyết: khi khách hàng báo request tạo task bị chậm, người vận hành phải tự tra log rời rạc trên từng service theo timestamp gần đúng và đoán mò, vì không có `trace_id` nào nối các log của cùng 1 request. Đây là lý do enhance cần distributed tracing.

```mermaid
sequenceDiagram
    actor Ops as Người vận hành
    participant GWLog as Log của api-gateway
    participant TSLog as Log của task-service
    participant NSLog as Log của notification-service
    participant SILog as Log của search-indexer

    Ops->>GWLog: Tìm log quanh thời điểm khách hàng báo lỗi
    GWLog-->>Ops: Thấy request POST /tasks lúc 10:00:01
    Ops->>TSLog: Tìm log task-service quanh 10:00:01, không có ID để lọc chính xác
    TSLog-->>Ops: Nhiều request khác cùng lúc, khó xác định đúng request nào
    Ops->>NSLog: Tìm log notification-service, đoán theo thời gian
    NSLog-->>Ops: Không chắc log này có phải cùng request hay không
    Ops->>SILog: Tìm log search-indexer, tiếp tục đoán
    SILog-->>Ops: Không xác định được service nào gây chậm
    Note over Ops: Không dựng lại được đường đi thật của request, khó xác định nghẽn ở đâu
```
