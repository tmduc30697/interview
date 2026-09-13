# Sequence Diagram — Enhance: Bulk Post Jobs

Đây là **enhance**, flow hoàn toàn mới: khi 1 công ty đăng hoặc gỡ hàng loạt tin cùng lúc, pipeline index phải xử lý đợt thay đổi lớn này mà không làm chậm việc cập nhật index cho các tin đăng bình thường của nhà tuyển dụng khác đang diễn ra song song.

```mermaid
sequenceDiagram
    actor Employer as Nhà tuyển dụng (đăng hàng loạt)
    actor OtherEmployer as Nhà tuyển dụng khác (đăng bình thường)
    participant JobSvc as Job Posting Service
    participant Queue as Index Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Job Search Index

    par Đăng hàng loạt
        Employer->>JobSvc: Đăng/gỡ hàng trăm tin cùng lúc
        JobSvc->>Queue: Publish nhiều INDEX_SYNC_EVENT, gắn lane = bulk
    and Đăng bình thường song song
        OtherEmployer->>JobSvc: Đăng 1 tin bình thường
        JobSvc->>Queue: Publish INDEX_SYNC_EVENT, lane = realtime
    end

    Queue->>Indexer: Route event theo lane riêng biệt, bulk không chặn realtime
    Indexer->>Index: Xử lý lane realtime ngay lập tức
    Index-->>Indexer: Tin của nhà tuyển dụng khác lên index ngay, không bị ảnh hưởng

    loop Xử lý lane bulk theo batch, throttled
        Indexer->>Index: Cập nhật index cho từng lô tin hàng loạt
        Index-->>Indexer: Batch indexed
    end

    Note over Queue,Indexer: Tách lane đảm bảo đợt đăng/gỡ hàng loạt không làm trễ index của các nhà tuyển dụng khác
```
