# Enhance sequence — Rolling restart toàn cluster giới hạn % kết nối reconnect đồng thời

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: rolling restart toàn cluster chat server sao cho tại mọi thời điểm, tổng số kết nối bị buộc reconnect đồng thời không vượt quá X% tổng số user online, tránh spike tải lên hệ thống presence/auth.

```mermaid
sequenceDiagram
    participant Orchestrator as Deploy Orchestrator
    participant Plan as CLUSTER_DRAIN_PLAN
    participant I1 as Instance 1 (5000 kết nối)
    participant I2 as Instance 2 (5000 kết nối)
    participant I3 as Instance 3 (5000 kết nối)
    participant Auth as Presence/Auth Service

    Orchestrator->>Plan: Bắt đầu rolling restart cluster (total_online_users=15000, max_percent_reconnect_simultaneous=10%)
    Note over Plan: Ngân sách reconnect đồng thời tối đa = 1500 kết nối tại 1 thời điểm

    Orchestrator->>Plan: Xin phép drain Instance 1 (5000 kết nối)
    Plan-->>Orchestrator: Từ chối drain toàn bộ cùng lúc, 5000 > ngân sách 1500

    Orchestrator->>Plan: Xin phép drain theo lô 1500 kết nối của Instance 1
    Plan-->>Orchestrator: OK, current_batch_reconnecting=1500
    Orchestrator->>I1: Drain lô 1 (1500 kết nối), gửi close_code=4000 cho từng kết nối trong lô
    I1->>Auth: 1500 client reconnect, xác thực lại
    Auth-->>I1: Xử lý xong, tải trong ngưỡng cho phép
    I1->>Plan: Lô 1 hoàn tất, current_batch_reconnecting=0

    Orchestrator->>Plan: Xin phép drain lô 2 (1500 kết nối tiếp theo của Instance 1)
    Plan-->>Orchestrator: OK
    Orchestrator->>I1: Drain lô 2
    I1->>Auth: 1500 client reconnect

    Note over Orchestrator,Plan: Lặp lại theo từng lô 1500 kết nối cho tới khi Instance 1 drain xong toàn bộ, rồi mới chuyển sang Instance 2 và Instance 3 theo cùng cơ chế, đảm bảo Auth/presence không bao giờ nhận quá 1500 lượt reconnect cùng lúc
```
