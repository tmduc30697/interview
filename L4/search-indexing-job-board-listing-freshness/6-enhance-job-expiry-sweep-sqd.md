# Sequence Diagram — Enhance: Job Expiry Sweep

Đây là **enhance**, flow hoàn toàn mới: tin có ngày hết hạn phải tự động bị gỡ khỏi index đúng thời điểm hết hạn, xử lý đúng múi giờ, mà không cần nhà tuyển dụng thao tác.

```mermaid
sequenceDiagram
    participant Scheduler as Expiry Sweep Scheduler
    participant DB as Job Postings DB
    participant Queue as Index Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Job Search Index

    loop Chạy liên tục theo mốc thời gian nhỏ, ví dụ mỗi phút
        Scheduler->>DB: Tìm job_posting có expires_at (quy đổi đúng timezone) đã tới hoặc vừa qua, status vẫn active
        DB-->>Scheduler: Danh sách tin vừa hết hạn

        loop for each job vừa hết hạn
            Scheduler->>DB: UPDATE job_postings SET status = expired
            Scheduler->>Queue: Publish INDEX_SYNC_EVENT (job_id, change_type = expired)
        end
    end

    Queue->>Indexer: Deliver expiry events
    Indexer->>Index: UPDATE job_index_document SET status = expired
    Index-->>Indexer: Indexed

    Note over Scheduler,Index: Mốc quét đủ nhỏ và tính đúng timezone để tin hết hạn không hiển thị quá lâu, và tin còn hạn không bị gỡ nhầm
```
