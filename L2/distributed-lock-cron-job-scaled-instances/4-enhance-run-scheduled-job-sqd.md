# Enhance sequence — Chỉ instance giữ lock mới chạy job, phải renew trước khi hết hạn

Đây là **enhance** của flow `run-scheduled-job` đã có ở base. So với base (mọi instance tự chạy job độc lập), enhance bắt buộc instance phải giành `JOB_LOCK` trước khi chạy, và phải renew (heartbeat) trước khi TTL hết nếu job chạy lâu hơn 1 lease. Đáp ứng yêu cầu 1: lock có TTL/lease rõ ràng, instance giữ lock phải renew trước khi TTL hết, nếu không renew kịp lock tự nhả cho instance khác.

```mermaid
sequenceDiagram
    participant PodA as Instance/Pod A
    participant PodB as Instance/Pod B
    participant Lock as Redis/ZooKeeper (JOB_LOCK)
    participant DB as Database / External System

    Note over PodA,PodB: Cả 2 pod cùng trigger cron gần như đồng thời

    PodA->>Lock: SET job-lock:send_daily_report NX PX 30000ms, fencing_token=N+1
    Lock-->>PodA: Giành lock thành công

    PodB->>Lock: SET job-lock:send_daily_report NX PX 30000ms
    Lock-->>PodB: Thất bại, lock đang bị PodA giữ
    PodB->>PodB: Skip, không chạy job

    PodA->>DB: Bắt đầu chạy job "send_daily_report"

    loop Job chạy lâu hơn 1 lease (vd mỗi 20s renew 1 lần)
        PodA->>Lock: EXPIRE job-lock:send_daily_report 30000ms (renew trước khi hết hạn)
        Lock-->>PodA: Renew thành công, vẫn giữ fencing_token=N+1
    end

    PodA->>DB: Job hoàn tất, ghi JOB_RUN (fencing_token_used=N+1, status=success)
    PodA->>Lock: DEL job-lock:send_daily_report (release ngay sau khi xong)

    Note over Lock: Nếu PodA không renew kịp trước khi TTL hết, lock tự hết hạn và instance khác có thể giành
```
