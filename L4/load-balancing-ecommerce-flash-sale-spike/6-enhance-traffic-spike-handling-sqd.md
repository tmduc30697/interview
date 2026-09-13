# Enhance sequence — Traffic spike handling (shed load có kiểm soát)

Đây là **enhance** của flow đã có ở base "Traffic spike handling". So với base, khi toàn cụm đạt ngưỡng tải tối đa, LB chủ động trả lỗi 503 kèm `Retry-After` cho một phần request thay vì để tất cả cùng dồn vào và timeout đồng loạt, đáp ứng **yêu cầu 2** (cơ chế shed load có kiểm soát khi cluster đạt ngưỡng tải tối đa).

```mermaid
sequenceDiagram
    actor Customers as Hàng loạt customer (traffic x20)
    participant LB as Load Balancer
    participant ShedController as Load Shed Controller
    participant Cluster as Toàn bộ Instance trong cụm

    Customers->>LB: Request đồng loạt tăng vọt
    LB->>ShedController: Kiểm tra cluster_load_snapshot hiện tại
    ShedController->>Cluster: Tổng hợp active_connections + latency toàn cụm
    Cluster-->>ShedController: Cụm đã đạt ngưỡng tải tối đa

    alt Request thuộc phần vượt ngưỡng chịu tải an toàn
        ShedController->>ShedController: Ghi LOAD_SHED_DECISION(decision=shed, retry_after_seconds=N)
        ShedController-->>LB: Yêu cầu từ chối request này
        LB-->>Customers: Trả 503 kèm Retry-After: N giây, không forward vào cụm
    else Request nằm trong ngưỡng cụm còn xử lý được
        ShedController->>ShedController: Ghi LOAD_SHED_DECISION(decision=serve)
        LB->>Cluster: Forward request bình thường
        Cluster-->>Customers: Xử lý và trả kết quả
    end
    Note over ShedController,Cluster: Một phần request bị từ chối có kiểm soát giúp phần còn lại vẫn được xử lý đúng hạn, thay vì toàn bộ cùng timeout
```
