# Enhance sequence — Fencing token chặn instance cũ ghi kết quả sau khi mất lock

Đây là **enhance**, flow hoàn toàn mới xử lý trường hợp instance bị GC pause/network delay tưởng mình còn giữ lock. Đáp ứng yêu cầu 3: chống trường hợp instance cũ tiếp tục ghi dữ liệu sau khi lock đã hết hạn và bị instance khác giành mất, bằng cách so sánh fencing token trước khi cho phép ghi kết quả job.

```mermaid
sequenceDiagram
    participant PodA as Instance/Pod A (bị GC pause)
    participant PodB as Instance/Pod B
    participant Lock as Redis/ZooKeeper (JOB_LOCK)
    participant JobSvc as Job Result Service
    participant DB as Database

    PodA->>Lock: SET job-lock:cleanup_old_data NX PX 30000ms, fencing_token=N+1
    Lock-->>PodA: Giành lock thành công

    PodA->>PodA: Bắt đầu chạy job, đang xử lý thì bị GC pause dài (hoặc network delay)

    Note over Lock: Trong lúc PodA bị pause, TTL hết hạn mà PodA không kịp renew

    PodB->>Lock: SET job-lock:cleanup_old_data NX PX 30000ms, fencing_token=N+2
    Lock-->>PodB: Giành lock thành công vì lock cũ đã hết hạn

    PodB->>DB: Chạy job, hoàn tất, gửi kết quả kèm fencing_token=N+2
    JobSvc->>JobSvc: Kiểm tra N+2 >= last_accepted_token (N+1), hợp lệ
    JobSvc->>DB: Ghi JOB_RUN (fencing_token_used=N+2, status=success)
    JobSvc->>JobSvc: Cập nhật last_accepted_token=N+2

    Note over PodA: PodA tỉnh lại sau GC pause, vẫn tưởng mình còn giữ lock với fencing_token=N+1
    PodA->>JobSvc: Gửi kết quả job (hoàn tất muộn) kèm fencing_token=N+1

    JobSvc->>JobSvc: Kiểm tra N+1 < last_accepted_token (N+2), token đã cũ
    JobSvc-->>PodA: Từ chối ghi, "fencing token expired, lock đã đổi chủ"
    JobSvc->>DB: Ghi LOCK_EVENT_LOG (event_type=fencing_rejected, fencing_token=N+1)

    Note over JobSvc,DB: Kết quả trễ của PodA không được phép ghi đè lên kết quả hợp lệ của PodB
```
