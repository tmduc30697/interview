# Sequence Diagram - Base: claim-and-process-job

Đây là flow **base**: worker định kỳ poll hàng đợi, thấy job đang ở trạng thái "AVAILABLE" thì tự cập nhật thẳng thành "PROCESSING" rồi xử lý, không qua bất kỳ lock/lease nào. Flow này là nền để so sánh với enhance, nơi bước "claim" sẽ được thay bằng thao tác atomic có TTL để tránh 2 worker cùng nhận 1 job.

```mermaid
sequenceDiagram
    participant W as Worker
    participant Q as Job Queue/DB

    W->>Q: Poll danh sách job status=AVAILABLE
    Q-->>W: Trả về job J1
    W->>Q: Update job J1 status=PROCESSING, assigned_worker_id=W
    Q-->>W: OK
    W->>W: Xử lý job J1 (đọc file, gửi email...)
    W->>Q: Update job J1 status=DONE
    Q-->>W: OK
```
