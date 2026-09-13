# Sequence Diagram - Base: create-task

Đây là flow **base**: request tạo task đi qua `api-gateway` -> `task-service` -> `notification-service` -> `search-indexer` (bất đồng bộ qua message queue), mỗi service chỉ ghi log cục bộ của riêng nó, không kèm bất kỳ ID chung nào. Flow này là nền để so sánh với enhance, nơi mỗi bước sẽ được gắn `trace_id`/`span` xuyên suốt.

```mermaid
sequenceDiagram
    actor Client
    participant GW as api-gateway
    participant TS as task-service
    participant MQ as Message Queue
    participant NS as notification-service
    participant SI as search-indexer

    Client->>GW: POST /tasks (tạo task mới)
    GW->>TS: Gọi tạo task
    TS->>TS: Ghi log cục bộ "task created"
    TS->>MQ: Publish event "task.created"
    TS-->>GW: 201 Created
    GW-->>Client: 201 Created
    MQ-->>NS: Deliver event "task.created"
    NS->>NS: Ghi log cục bộ "gửi notification"
    MQ-->>SI: Deliver event "task.created"
    SI->>SI: Ghi log cục bộ "index task vào search"
```
