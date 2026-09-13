# Base sequence — Route request (round-robin, không phân biệt tenant)

Đây là **base**, flow route request hiện tại: gateway chọn backend instance bằng round-robin/least-connections, hoàn toàn không biết và không quan tâm request đến từ tenant nào. Flow này là tiền đề để so sánh với yêu cầu 1 và yêu cầu 4 của đề bài (giới hạn và trọng số theo tenant).

```mermaid
sequenceDiagram
    actor Tenant as Tenant bất kỳ
    participant GW as Gateway Instance
    participant BE as Backend Instance

    Tenant->>GW: Gửi request kèm API key
    GW->>GW: Xác thực tenant qua API key (chỉ để biết danh tính, không dùng để giới hạn)
    GW->>GW: Chọn backend theo round-robin/least-connections
    GW->>BE: Forward request
    BE-->>GW: Response
    GW-->>Tenant: Trả response
    Note over GW,BE: Không có bước kiểm tra giới hạn tài nguyên nào theo tenant trước khi forward
```
