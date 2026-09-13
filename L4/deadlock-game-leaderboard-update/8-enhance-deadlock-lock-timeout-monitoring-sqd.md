# Enhance sequence — Đo và log tỷ lệ deadlock/lock-timeout theo giờ cao điểm

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Đáp ứng yêu cầu 5 của đề bài: đo và log tỷ lệ deadlock/lock-timeout theo giờ để xác định có cần điều chỉnh tần suất chạy job rank hay không.

```mermaid
sequenceDiagram
    participant App as Learning App
    participant Job as Rank Recalculation Job
    participant Stats as DEADLOCK_STATS
    participant Ops as Đội vận hành

    loop Mỗi lần transaction cộng điểm gặp deadlock hoặc lock timeout
        App->>Stats: Tăng deadlock_count/lock_timeout_count cho hour_window hiện tại, context=score_update
    end

    loop Mỗi lần batch của Rank Job gặp deadlock hoặc lock timeout
        Job->>Stats: Tăng deadlock_count/lock_timeout_count cho hour_window hiện tại, context=rank_job_batch
    end

    Ops->>Stats: Truy vấn tỷ lệ deadlock/lock-timeout theo từng giờ trong ngày
    Stats-->>Ops: Trả về báo cáo, ví dụ giờ cao điểm 19h-21h có tỷ lệ deadlock cao gấp 5 lần giờ thường

    alt Tỷ lệ deadlock/lock-timeout ở giờ cao điểm vượt ngưỡng chấp nhận được
        Ops->>Job: Điều chỉnh giảm tần suất chạy job rank trong khung giờ cao điểm, hoặc giảm kích thước batch
    else Tỷ lệ vẫn trong ngưỡng bình thường
        Ops->>Job: Giữ nguyên cấu hình hiện tại
    end
```
