# Sequence Diagram - Enhance: dead-job-scan

Đây là flow **enhance mới**: một tiến trình quét định kỳ (dead job scanner) chủ động rà soát các job đang ở trạng thái "PROCESSING" quá lâu so với TTL kỳ vọng, thay vì chỉ dựa hoàn toàn vào cơ chế TTL tự nhiên - dùng để phát hiện sớm rò rỉ lock hoặc lỗi hệ thống (ví dụ TTL không tự expire được do bug, clock skew). Đáp ứng yêu cầu 5.

```mermaid
sequenceDiagram
    participant Scanner as Dead Job Scanner (chạy định kỳ)
    participant Q as Job Queue/DB
    participant Alert as Alerting/Ops

    loop Mỗi chu kỳ quét (ví dụ mỗi 30s)
        Scanner->>Q: Tìm job status=PROCESSING có updated_at quá lâu so với lease_ttl_seconds kỳ vọng
        Q-->>Scanner: Danh sách job nghi ngờ (ví dụ J5 đã PROCESSING 10 phút dù TTL=60s)
        alt Lease của job đã thực sự hết hạn nhưng job chưa được trả về AVAILABLE
            Scanner->>Q: Ghi log DEAD_JOB_SCAN_LOG, cưỡng chế trả job J5 về status=AVAILABLE
            Q-->>Scanner: OK
            Scanner->>Alert: Cảnh báo rò rỉ lock/lỗi hệ thống cho người vận hành
        else Lease vẫn còn hạn, chỉ là job xử lý lâu hợp lệ
            Scanner->>Q: Ghi log DEAD_JOB_SCAN_LOG với action_taken=no-op, không can thiệp
        end
    end
```
