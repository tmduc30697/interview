# Enhance sequence — Giám sát usage gần thời gian thực và cảnh báo sớm

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không đo lường usage tổng, mỗi service không biết tình trạng của các service khác). Đáp ứng yêu cầu 5 của đề bài: giám sát usage thực tế so với hạn mức đối tác, cảnh báo sớm khi xu hướng traffic tiến gần ngưỡng trước khi bị chặn hoàn toàn.

```mermaid
sequenceDiagram
    participant Throttle as Outbound Throttle Gateway
    participant Window as PARTNER_USAGE_WINDOW
    participant Monitor as Usage Monitor (job định kỳ)
    participant Alert as USAGE_ALERT
    actor OnCall as Đội trực (on-call)

    loop Mỗi lần gửi SMS thành công
        Throttle->>Window: Ghi nhận request_count vào window hiện tại
    end

    loop Định kỳ (ví dụ mỗi vài giây)
        Monitor->>Window: Đọc request_count và max_requests_per_second gần thời gian thực
        Monitor->>Monitor: Tính utilization_pct = request_count / max_requests_per_second

        alt utilization_pct vượt ngưỡng cảnh báo, ví dụ 80%
            Monitor->>Alert: Tạo USAGE_ALERT(status=open, threshold_pct, triggered_at)
            Alert-->>OnCall: Thông báo sớm, ví dụ do một service lỗi gửi lặp

            Note over OnCall,Alert: Đội trực có thể xác định service nào đang chiếm phần lớn quota và can thiệp trước khi đối tác chặn cả tài khoản

        else utilization_pct trở lại bình thường
            Monitor->>Alert: Đánh dấu USAGE_ALERT liên quan là status=resolved
        end
    end
```
