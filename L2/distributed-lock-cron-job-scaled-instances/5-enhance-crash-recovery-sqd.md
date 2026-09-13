# Enhance sequence — Instance crash giữa job, lock tự hết hạn, instance khác chạy lại

Đây là **enhance**, flow hoàn toàn mới mô tả trường hợp instance giữ lock crash bất ngờ giữa lúc chạy job, không kịp release lock. Đáp ứng yêu cầu 2: lock phải tự hết hạn sau TTL và job có thể chạy lại bởi instance khác, không bị khóa chết vĩnh viễn dù có 1 instance chết đột ngột.

```mermaid
sequenceDiagram
    participant PodA as Instance/Pod A (sẽ crash)
    participant PodB as Instance/Pod B
    participant Lock as Redis/ZooKeeper (JOB_LOCK)
    participant DB as Database / External System

    PodA->>Lock: SET job-lock:cleanup_old_data NX PX 30000ms, fencing_token=N+1
    Lock-->>PodA: Giành lock thành công

    PodA->>DB: Bắt đầu chạy job "cleanup_old_data"

    Note over PodA: Instance A crash đột ngột (OOM kill, pod bị evict...), không kịp renew hay release lock

    PodB->>Lock: SET job-lock:cleanup_old_data NX PX 30000ms
    Lock-->>PodB: Thất bại, lock vẫn đang được PodA giữ (chưa hết TTL)
    PodB->>PodB: Skip lần trigger này

    Note over Lock: TTL 30s trôi qua mà không có renew nào từ PodA, lock tự hết hạn

    PodB->>Lock: (lần trigger kế tiếp) SET job-lock:cleanup_old_data NX PX 30000ms, fencing_token=N+2
    Lock-->>PodB: Giành lock thành công vì lock cũ đã hết hạn

    PodB->>DB: Chạy lại job "cleanup_old_data" từ đầu
    PodB->>DB: Ghi JOB_RUN (instance_id=PodB, fencing_token_used=N+2, status=success)
    PodB->>Lock: DEL job-lock:cleanup_old_data

    Note over Lock,PodB: Job không bị khóa chết vĩnh viễn dù PodA đã chết, hệ thống tự phục hồi sau tối đa 1 TTL
```
