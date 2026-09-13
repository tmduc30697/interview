# Sequence Diagram — Base: Post Job

Đây là **base**, flow "đăng tin tuyển dụng" — cho thấy tin mới chỉ vào index sau chu kỳ đồng bộ định kỳ (batch), là tiền đề cho vấn đề độ trễ mà enhance phải giải quyết ở cả chiều đăng lẫn chiều đóng tin.

```mermaid
sequenceDiagram
    actor Employer
    participant JobSvc as Job Posting Service
    participant DB as Job Postings DB
    participant BatchSync as Periodic Sync Job (batch)
    participant Index as Job Search Index

    Employer->>JobSvc: Đăng tin tuyển dụng mới
    JobSvc->>DB: INSERT INTO job_postings (status = active)
    DB-->>JobSvc: job_id created
    JobSvc-->>Employer: Tin đã đăng

    Note over DB,BatchSync: Tin chưa xuất hiện trong kết quả tìm kiếm ngay
    BatchSync->>DB: Chạy theo chu kỳ định kỳ, quét các thay đổi
    BatchSync->>Index: Đồng bộ tin mới vào index
    Index-->>BatchSync: Indexed

    Note over BatchSync,Index: Có độ trễ giữa lúc đăng tin và lúc tin xuất hiện trong tìm kiếm, phụ thuộc chu kỳ batch
```
