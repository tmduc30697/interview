# Cron/scheduled job chạy trên nhiều instance service (horizontally scaled)

**Hệ thống:** Một service backend được scale ra nhiều instance (pod) để chịu tải, nhưng có vài job định kỳ (gửi báo cáo, dọn dữ liệu) chỉ được chạy đúng 1 lần mỗi chu kỳ.

**Vai trò của flow:** Dùng distributed lock (lease-based, TTL) trên Redis/ZooKeeper để đảm bảo chỉ một instance giành được lock mới được chạy job, các instance khác skip.

**Yêu cầu cụ thể:**
- Lock phải có TTL (lease) rõ ràng; instance giữ lock phải renew (heartbeat) trước khi TTL hết, nếu không renew kịp thì lock tự nhả cho instance khác.
- Xử lý crash: nếu instance giữ lock crash giữa job (không kịp release lock), lock phải tự hết hạn sau TTL và job có thể chạy lại bởi instance khác — không bị "khoá chết" vĩnh viễn.
- Chống trường hợp instance cũ tưởng mình còn giữ lock (do GC pause/network delay) tiếp tục ghi dữ liệu sau khi lock đã hết hạn và bị instance khác giành mất (fencing token / lock version check trước khi ghi kết quả job).
- Log rõ instance nào giữ lock, thời điểm acquire/release/renew, để debug khi nghi ngờ job chạy trùng.
- Đo SLA: thời gian tối đa từ lúc lock cũ hết hạn tới lúc instance mới giành được lock mới (failover time) phải dưới một ngưỡng cụ thể (ví dụ 10s).

---

# Base ERD — Job định kỳ chạy độc lập trên từng instance, chưa có điều phối

Đây là **base**: trạng thái hệ thống *trước khi* có distributed lock. Suy luận từ đề bài, base đã có `JOB_DEFINITION` mô tả job định kỳ (tên, biểu thức cron/lịch chạy), và mỗi lần job được kích hoạt tạo 1 `JOB_RUN` ghi lại instance nào chạy, kết quả ra sao. Vì hệ thống được scale ra nhiều instance chạy cùng một lịch cron nội bộ, base chưa có cơ chế nào đảm bảo chỉ 1 instance thực thi — mỗi instance tự trigger job theo timer riêng của mình.

```mermaid
erDiagram
    JOB_DEFINITION ||--o{ JOB_RUN : "được thực thi thành"

    JOB_DEFINITION {
        string id PK
        string name "vd send_daily_report, cleanup_old_data"
        string cron_expression
    }
    JOB_RUN {
        string id PK
        string job_definition_id FK
        string instance_id "pod nào chạy"
        datetime started_at
        datetime finished_at
        string status "success|failed"
    }
```

---

# Base sequence — Mỗi instance tự chạy job theo lịch riêng, chạy trùng lặp

Đây là **base**, flow chạy job định kỳ hiện tại: mỗi instance/pod có timer nội bộ riêng theo `cron_expression`, khi tới giờ thì tự chạy job mà không hỏi ý kiến ai. Đây chính là vấn đề nêu trong đề bài — service được scale ra nhiều instance để chịu tải, nhưng job như gửi báo cáo hay dọn dữ liệu lại vô tình chạy nhiều lần trùng lặp mỗi chu kỳ, có thể gây gửi báo cáo trùng hoặc dọn dữ liệu race nhau.

```mermaid
sequenceDiagram
    participant PodA as Instance/Pod A
    participant PodB as Instance/Pod B
    participant PodC as Instance/Pod C
    participant DB as Database / External System

    Note over PodA,PodC: Cả 3 pod đều có cùng cron_expression, timer nội bộ kích hoạt cùng lúc

    par Cả 3 instance cùng trigger job đồng thời
        PodA->>PodA: Cron trigger "send_daily_report"
        PodB->>PodB: Cron trigger "send_daily_report"
        PodC->>PodC: Cron trigger "send_daily_report"
    end

    par Không ai biết ai cũng đang chạy, cả 3 đều thực thi
        PodA->>DB: Chạy job, gửi báo cáo, ghi JOB_RUN
        PodB->>DB: Chạy job, gửi báo cáo, ghi JOB_RUN
        PodC->>DB: Chạy job, gửi báo cáo, ghi JOB_RUN
    end

    Note over DB: Báo cáo bị gửi 3 lần trùng lặp, hoặc dữ liệu bị dọn 3 lần race nhau
```

---

# Enhance ERD — Thêm lease-based lock, fencing token và log điều phối

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có thêm 2 entity mới và `JOB_RUN` được bổ sung trường ràng buộc với lock, ứng trực tiếp với các yêu cầu:

- `JOB_LOCK` (mới) với `holder_instance_id`, `expires_at`, `fencing_token` — đáp ứng yêu cầu 1 (lock có TTL/lease rõ ràng, holder phải renew trước khi hết hạn) và yêu cầu 2 (nếu holder crash không renew kịp, lock tự hết hạn cho instance khác giành).
- `JOB_RUN.fencing_token` — đáp ứng yêu cầu 3 (job service phải kiểm tra fencing token khi ghi kết quả, từ chối ghi nếu token cũ đã bị vô hiệu do lock đã đổi chủ).
- `LOCK_EVENT_LOG` (mới) — đáp ứng yêu cầu 4 (log rõ instance nào giữ lock, thời điểm acquire/release/renew để debug) và là nguồn dữ liệu tính `failover_time_ms` cho yêu cầu 5 (SLA failover dưới ngưỡng, vd 10s).

```mermaid
erDiagram
    JOB_DEFINITION ||--o{ JOB_RUN : "được thực thi thành"
    JOB_DEFINITION ||--o| JOB_LOCK : "có 1 lock hiện hành"
    JOB_DEFINITION ||--o{ LOCK_EVENT_LOG : "lịch sử điều phối"

    JOB_DEFINITION {
        string id PK
        string name "vd send_daily_report, cleanup_old_data"
        string cron_expression
        int lease_ttl_seconds "vd 30"
    }
    JOB_LOCK {
        string job_definition_id PK "1 lock hiện hành mỗi job"
        string holder_instance_id
        int fencing_token "tăng dần mỗi lần acquire mới"
        datetime acquired_at
        datetime expires_at
    }
    JOB_RUN {
        string id PK
        string job_definition_id FK
        string instance_id "pod nào chạy"
        int fencing_token_used
        datetime started_at
        datetime finished_at
        string status "success|failed|rejected_stale_token"
    }
    LOCK_EVENT_LOG {
        string id PK
        string job_definition_id FK
        string instance_id
        string event_type "acquire|renew|release|expire|fencing_rejected"
        int fencing_token
        datetime occurred_at
    }
```

---

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

---

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

---

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

---

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
