# Sequence Diagram - Base: worker-crash-job-stuck

Đây là flow **base** minh hoạ vấn đề chính mà đề bài cần giải quyết: khi worker đang xử lý job mà crash (hoặc bị autoscale kill) giữa chừng, job vẫn nằm ở trạng thái "PROCESSING" mãi mãi vì không có TTL/lease nào theo dõi, không worker nào khác dám nhận lại, job bị kẹt vĩnh viễn. Đây là lý do enhance cần TTL tự động trả job về "AVAILABLE" và cơ chế dead job detection.

```mermaid
sequenceDiagram
    participant W as Worker (sắp crash)
    participant Q as Job Queue/DB
    participant W2 as Worker khác

    W->>Q: Update job J1 status=PROCESSING, assigned_worker_id=W
    Q-->>W: OK
    Note over W: Worker bị crash/bị autoscale kill giữa lúc xử lý
    W2->>Q: Poll danh sách job status=AVAILABLE
    Q-->>W2: Không thấy J1 (vẫn đang ở PROCESSING)
    Note over Q: Không có TTL, không ai phát hiện worker W đã chết, J1 kẹt vĩnh viễn ở PROCESSING
```
