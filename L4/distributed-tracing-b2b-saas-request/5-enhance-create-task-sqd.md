# Sequence Diagram - Enhance: create-task

Đây là flow **enhance** của `create-task` (so với base ở file `2-base-create-task-sqd.md`): `api-gateway` sinh `trace_id` nếu client chưa gửi, mỗi service tạo span con gắn `parent_span_id` đúng (kể cả khi `notification-service` và `search-indexer` được gọi song song qua message queue), và `trace_id`/`parent_span_id` được propagate qua cả header HTTP lẫn header của message queue. Đáp ứng yêu cầu 1 (propagate trace_id qua header, kể cả async qua MQ) và yêu cầu 2 (mỗi service tạo span con đúng quan hệ cha-con, kể cả gọi song song).

```mermaid
sequenceDiagram
    actor Client
    participant GW as api-gateway
    participant TS as task-service
    participant MQ as Message Queue
    participant NS as notification-service
    participant SI as search-indexer

    Client->>GW: POST /tasks (không kèm trace_id)
    GW->>GW: Sinh trace_id=T1, tạo span S1 (parent=null)
    GW->>TS: Gọi tạo task, header: trace_id=T1, parent_span_id=S1
    TS->>TS: Tạo span S2 (trace_id=T1, parent_span_id=S1)
    TS->>MQ: Publish event "task.created", header: trace_id=T1, parent_span_id=S2
    TS-->>GW: 201 Created, kết thúc span S2
    GW-->>Client: 201 Created, kết thúc span S1
    par Xử lý song song từ message queue
        MQ-->>NS: Deliver event, header trace_id=T1, parent_span_id=S2
        NS->>NS: Tạo span S3 (trace_id=T1, parent_span_id=S2), gửi notification
    and
        MQ-->>SI: Deliver event, header trace_id=T1, parent_span_id=S2
        SI->>SI: Tạo span S4 (trace_id=T1, parent_span_id=S2), index task
    end
    Note over TS,SI: Cây span dựng lại được: S1 -> S2 -> (S3, S4) chạy song song
```
