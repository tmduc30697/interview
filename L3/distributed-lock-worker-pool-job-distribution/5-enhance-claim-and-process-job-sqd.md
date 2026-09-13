# Sequence Diagram - Enhance: claim-and-process-job

Đây là flow **enhance** của `claim-and-process-job` (so với base ở file `2-base-claim-and-process-job-sqd.md`): 2 worker cùng thấy job "AVAILABLE" và cùng cố acquire lock/lease gần như đồng thời (race condition khi liệt kê job không atomic với acquire), nhưng thao tác claim thực tế là 1 lệnh atomic (compare-and-swap trên trạng thái job hoặc acquire trên coordinator) nên chỉ đúng 1 worker thắng và có TTL ngay từ lúc claim. Đáp ứng yêu cầu 1 (claim bằng lock/lease có TTL) và yêu cầu 2 (race condition chỉ 1 worker thắng, worker thua nhận biết ngay).

```mermaid
sequenceDiagram
    participant W1 as Worker 1
    participant W2 as Worker 2
    participant Q as Job Queue/DB (atomic claim)

    W1->>Q: Poll job status=AVAILABLE
    Q-->>W1: Trả về job J1
    W2->>Q: Poll job status=AVAILABLE (gần như cùng lúc)
    Q-->>W2: Trả về job J1
    par Cả 2 worker cùng cố claim J1
        W1->>Q: Atomic claim J1 (CAS status=AVAILABLE->PROCESSING, set lease_ttl=60s)
        W2->>Q: Atomic claim J1 (CAS status=AVAILABLE->PROCESSING, set lease_ttl=60s)
    end
    Q-->>W1: Thành công, cấp fencing_token=101, expires_at=+60s
    Q-->>W2: Thất bại, job đã ở PROCESSING (thua race)
    W2->>W2: Nhận biết ngay, không xử lý song song, quay lại poll job khác
    W1->>W1: Xử lý job J1
    W1->>Q: Update job J1 status=DONE, release claim
    Q-->>W1: OK
```
