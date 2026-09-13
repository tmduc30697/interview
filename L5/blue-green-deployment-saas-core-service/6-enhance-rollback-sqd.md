# Enhance sequence — Rollback (tự động, theo tiêu chí lỗi 5xx)

Đây là **enhance**, cùng flow "Rollback" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu thứ 4 của đề bài: có `ROLLBACK_POLICY` định nghĩa rõ tiêu chí (tỷ lệ lỗi 5xx vượt ngưỡng X% trong Y phút), và giám sát chạy tự động ngay sau khi router chuyển traffic — không chờ user report như base.

```mermaid
sequenceDiagram
    participant Router as Router/Load Balancer
    participant Monitor as Auto Rollback Monitor
    participant Policy as ROLLBACK_POLICY store
    participant Blue as Blue Deployment (vẫn đang chạy, chưa tắt)
    participant Green as Green Deployment (vừa nhận traffic)

    Note over Router: Ngay sau ROUTER_SWITCH_EVENT chuyển traffic sang Green
    Monitor->>Policy: Lấy error_rate_threshold + window_minutes cho service
    loop Trong suốt window_minutes ngay sau switch
        Monitor->>Green: Theo dõi tỷ lệ lỗi 5xx theo thời gian thực
        alt Tỷ lệ lỗi vượt threshold
            Monitor->>Router: Kích hoạt rollback ngay (không chờ phát hiện thủ công)
            Router->>Router: Tạo ROLLBACK_EVENT (triggered_by=auto_policy), revert target về Blue
            Router-->>Blue: Traffic quay lại Blue (Blue vẫn đang chạy sẵn, không cần khởi động lại)
            Green->>Green: status=rolled_back
            Monitor-->>Monitor: Dừng theo dõi, cảnh báo on-call để điều tra thêm
        else Tỷ lệ lỗi trong ngưỡng cho phép
            Monitor->>Monitor: Tiếp tục theo dõi tới hết window
        end
    end
    Note over Monitor: Hết window mà không vượt threshold → Green được coi là ổn định, Blue chuyển sang retiring
```
