# Enhance sequence — Connection counter race condition

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — base đọc/ghi `active_connections` một cách đơn giản không đảm bảo atomic. Đáp ứng **yêu cầu 4** (nhiều request cùng cập nhật bộ đếm connection/tải của một instance đồng thời, đảm bảo số liệu dùng để ra quyết định route không bị lệch do cập nhật không atomic).

```mermaid
sequenceDiagram
    actor RequestX as Request X
    actor RequestY as Request Y (gần như đồng thời)
    participant LB as Load Balancer
    participant Counter as CONNECTION_COUNTER_UPDATE (atomic, có version)
    participant InstanceA as Instance A

    par Hai request tới gần như cùng lúc
        RequestX->>LB: Route tới InstanceA
        LB->>Counter: Ghi delta=+1 (atomic increment), lấy resulting_version
        Counter-->>LB: active_connections cập nhật thành N+1, version tăng đúng 1 nấc
    and
        RequestY->>LB: Route tới InstanceA
        LB->>Counter: Ghi delta=+1 (atomic increment), lấy resulting_version
        Counter-->>LB: active_connections cập nhật thành N+2, version tăng đúng 1 nấc tiếp theo
    end
    Note over Counter,InstanceA: Nhờ increment atomic thay vì đọc-cộng-ghi riêng lẻ, active_connections phản ánh đúng N+2, không bị mất một trong hai lần cập nhật

    RequestX->>InstanceA: Xử lý xong, trả kết quả
    LB->>Counter: Ghi delta=-1 (atomic decrement)
    Counter-->>LB: active_connections giảm đúng về N+1
    Note over LB,Counter: Quyết định route tiếp theo dựa trên số liệu chính xác, không bị lệch do race condition giữa các lần cập nhật đồng thời
```
