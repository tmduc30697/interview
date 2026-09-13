# Enhance sequence — Test toàn trình rolling restart 5 instance dưới traffic liên tục

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: kịch bản test tự động rolling restart lần lượt 5 instance của một service trong lúc có traffic liên tục chạy nền, đo tỷ lệ lỗi/timeout toàn bộ quá trình và so sánh với ngưỡng SLA đã định nghĩa trước.

```mermaid
sequenceDiagram
    participant Test as Test Orchestrator
    participant LoadGen as Load Generator (traffic nền)
    participant GW as API Gateway
    participant RR as ROLLING_RESTART_RUN
    participant Cluster as Cluster 5 instance (Service X)

    Test->>RR: INSERT ROLLING_RESTART_RUN(total_instances=5, status=running)
    Test->>LoadGen: Bắt đầu bắn traffic liên tục qua GW
    LoadGen->>GW: Request liên tục trong suốt quá trình test

    loop Với từng instance trong 5 instance (tuần tự)
        Test->>Cluster: Gửi SIGTERM cho instance kế tiếp
        Cluster->>GW: DRAIN_SIGNAL(instance=kế tiếp)
        GW->>GW: Loại instance khỏi pool, đóng keep-alive cũ, route tiếp qua 4 instance còn lại
        Cluster->>Cluster: Instance restart xong, đăng ký lại vào pool (status=healthy)
        Test->>RR: UPDATE instances_restarted += 1
    end

    LoadGen->>Test: Dừng traffic, tổng hợp request_count và error_count (timeout/5xx)
    Test->>RR: UPDATE request_count, error_count
    Test->>Test: Tính tỷ lệ lỗi = error_count / request_count

    alt Tỷ lệ lỗi <= sla_error_threshold đã định nghĩa trước
        Test->>RR: UPDATE status=completed
        Test-->>Test: Test PASS
    else Tỷ lệ lỗi vượt ngưỡng SLA
        Test->>RR: UPDATE status=failed
        Test-->>Test: Test FAIL, đính kèm log các request lỗi để điều tra
    end
```
