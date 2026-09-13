# Sequence Diagram — Base: Discover Instance

Đây là **base**, flow một service khác gọi thẳng tới registry mỗi lần cần biết danh sách instance — chưa có cache, chưa có cơ chế TTL/invalidate, đây chính là điểm sẽ thay đổi ở enhance.

```mermaid
sequenceDiagram
    actor Caller as Calling Service
    participant Registry as Service Registry
    participant Instance as Target Instance

    Caller->>Registry: Get instances of Service X
    Registry-->>Caller: [Instance list]
    Caller->>Instance: Call instance directly
    Instance-->>Caller: Response
```
