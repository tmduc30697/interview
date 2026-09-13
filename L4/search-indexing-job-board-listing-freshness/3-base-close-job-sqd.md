# Sequence Diagram — Base: Close Job

Đây là **base**, flow "đóng tin đã tuyển đủ" — vấn đề cốt lõi mà enhance phải giải quyết: tin bị đóng trong DB nhưng vẫn hiển thị trong kết quả tìm kiếm cho tới chu kỳ đồng bộ định kỳ kế tiếp, khiến ứng viên vẫn nộp hồ sơ vào tin đã đóng.

```mermaid
sequenceDiagram
    actor Employer
    participant JobSvc as Job Posting Service
    participant DB as Job Postings DB
    participant BatchSync as Periodic Sync Job (batch)
    participant Index as Job Search Index
    actor Candidate

    Employer->>JobSvc: Đánh dấu tin đã tuyển đủ / đóng tin
    JobSvc->>DB: UPDATE job_postings SET status = closed
    DB-->>JobSvc: Updated
    JobSvc-->>Employer: Tin đã đóng

    Note over Index: Index chưa được cập nhật, tin vẫn còn status = active trong index
    Candidate->>Index: Tìm kiếm việc làm
    Index-->>Candidate: Tin đã đóng vẫn xuất hiện trong kết quả (chưa tới chu kỳ sync)

    BatchSync->>DB: Chạy theo chu kỳ định kỳ
    BatchSync->>Index: Đồng bộ status = closed
    Index-->>BatchSync: Indexed, tin biến mất khỏi kết quả từ lúc này
```
