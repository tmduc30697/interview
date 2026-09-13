# Enhance sequence — Breaker metrics & alerting

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base. Đáp ứng yêu cầu thứ 5 của đề bài: đo tỉ lệ fail-fast, số lần breaker chuyển trạng thái trong ngày, và alert khi breaker mở quá lâu.

```mermaid
sequenceDiagram
    participant Breaker as CIRCUIT_BREAKER_STATE
    participant App as Checkout Service
    participant Metric as BREAKER_METRIC store
    participant AlertJob as Alert Monitor
    participant Alert as BREAKER_ALERT store
    actor OnCall as On-call Engineer

    App->>Metric: Mỗi lần fail-fast do breaker open, tăng fail_fast_count trong window
    Breaker->>Metric: Mỗi lần chuyển state (closed→open, open→half_open, half_open→closed/open), tăng state_transition_count

    loop Định kỳ theo dõi
        AlertJob->>Breaker: Kiểm tra state hiện tại và opened_at
        alt state=open và đã mở quá lâu (vượt ngưỡng thời gian cho phép)
            AlertJob->>Alert: Ghi BREAKER_ALERT(duration_ms, alert_triggered_at)
            Alert-->>OnCall: Cảnh báo sự cố kéo dài ở phía gateway, cần điều tra
        else Trong ngưỡng bình thường
            AlertJob->>AlertJob: Không làm gì thêm
        end
    end

    Metric-->>OnCall: Báo cáo cuối ngày: tỉ lệ request bị fail-fast, số lần breaker đổi trạng thái trong ngày
```
