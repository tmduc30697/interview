# Enhance sequence — Log điều phối lock và đo SLA failover time

Đây là **enhance**, flow hoàn toàn mới mô tả việc ghi log chi tiết mọi thao tác lock và tính toán chỉ số SLA từ log đó. Đáp ứng yêu cầu 4 (log rõ instance nào giữ lock, thời điểm acquire/release/renew để debug khi nghi ngờ job chạy trùng) và yêu cầu 5 (đo thời gian tối đa từ lúc lock cũ hết hạn tới lúc instance mới giành được lock mới phải dưới ngưỡng, vd 10s).

```mermaid
sequenceDiagram
    participant PodA as Instance/Pod A
    participant PodB as Instance/Pod B
    participant Lock as Redis/ZooKeeper (JOB_LOCK)
    participant LogDB as LOCK_EVENT_LOG
    participant Job as SLA Monitoring Job
    actor Dev as Dev/SRE

    PodA->>Lock: Acquire lock, fencing_token=N+1
    Lock-->>PodA: Thành công
    PodA->>LogDB: INSERT LOCK_EVENT_LOG (instance_id=PodA, event_type=acquire, fencing_token=N+1, occurred_at=t0)

    PodA->>Lock: Renew lock (heartbeat)
    Lock-->>PodA: Thành công
    PodA->>LogDB: INSERT LOCK_EVENT_LOG (event_type=renew, occurred_at=t1)

    Note over PodA: PodA crash, không renew kịp, lock hết hạn tại t_expire

    Lock->>LogDB: INSERT LOCK_EVENT_LOG (instance_id=PodA, event_type=expire, occurred_at=t_expire)

    PodB->>Lock: Acquire lock, fencing_token=N+2
    Lock-->>PodB: Thành công tại t_acquire_new
    PodB->>LogDB: INSERT LOCK_EVENT_LOG (instance_id=PodB, event_type=acquire, fencing_token=N+2, occurred_at=t_acquire_new)

    Job->>LogDB: SELECT event log của job này, đối chiếu t_expire và t_acquire_new gần nhất
    LogDB-->>Job: t_expire=..., t_acquire_new=...
    Job->>Job: failover_time_ms = t_acquire_new - t_expire

    alt failover_time_ms dưới ngưỡng SLA (vd < 10s)
        Job-->>Dev: OK, đạt SLA failover
    else failover_time_ms vượt ngưỡng
        Job-->>Dev: Cảnh báo vi phạm SLA failover, kèm chi tiết log để điều tra
    end

    Dev->>LogDB: Khi nghi ngờ job chạy trùng, truy vấn LOCK_EVENT_LOG theo job_definition_id để xem instance nào giữ lock tại thời điểm nghi vấn
```
