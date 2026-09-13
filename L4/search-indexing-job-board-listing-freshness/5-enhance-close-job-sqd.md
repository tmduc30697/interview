# Sequence Diagram — Enhance: Close Job

Đây là **enhance**, flow "đóng tin" đã tồn tại ở base ([3-base-close-job-sqd.md](3-base-close-job-sqd.md)) nay thay đổi cốt lõi: thay vì chờ chu kỳ batch, việc đóng tin phát ngay `INDEX_SYNC_EVENT` và index được cập nhật tức thời, để tin biến mất khỏi kết quả tìm kiếm ngay lập tức.

```mermaid
sequenceDiagram
    actor Employer
    participant JobSvc as Job Posting Service
    participant DB as Job Postings DB
    participant Queue as Index Sync Event Queue
    participant Indexer as Index Sync Worker
    participant Index as Job Search Index
    actor Candidate

    Employer->>JobSvc: Đánh dấu tin đã tuyển đủ / đóng tin
    JobSvc->>DB: UPDATE job_postings SET status = closed
    DB-->>JobSvc: Updated
    JobSvc->>Queue: Publish INDEX_SYNC_EVENT (job_id, change_type = closed), priority cao

    Queue->>Indexer: Deliver event ngay lập tức
    Indexer->>Index: UPDATE job_index_document SET status = closed
    Index-->>Indexer: Indexed

    JobSvc-->>Employer: Tin đã đóng
    Candidate->>Index: Tìm kiếm việc làm
    Index-->>Candidate: Tin đã đóng không còn xuất hiện, gần như ngay sau khi đóng
```
