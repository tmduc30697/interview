# Enhance sequence — Toàn cụm nhận tín hiệu shutdown gần đồng thời, vẫn giữ worker tiếp nhận việc mới

Đây là flow **enhance** hoàn toàn mới so với base, đáp ứng **yêu cầu 5** của đề bài: khi rolling deploy khiến toàn bộ worker trong cụm nhận tín hiệu shutdown gần như đồng thời, `CLUSTER_DRAIN_PLAN` đảm bảo luôn còn tối thiểu một số worker tiếp tục nhận message mới trong lúc số còn lại đang drain, tránh hàng đợi bị dồn ứ không ai xử lý.

```mermaid
sequenceDiagram
    participant Orchestrator as Deploy Orchestrator
    participant Plan as CLUSTER_DRAIN_PLAN
    participant W1 as Worker 1
    participant W2 as Worker 2
    participant W3 as Worker 3
    participant Queue as Message Queue

    Orchestrator->>Plan: Bắt đầu rolling deploy cụm (total_workers=3, min_active_workers=1)
    Orchestrator->>W1: Gửi tín hiệu shutdown
    Orchestrator->>W2: Gửi tín hiệu shutdown
    Orchestrator->>W3: Gửi tín hiệu shutdown

    W1->>Plan: Xin phép chuyển sang draining
    Plan-->>W1: OK (workers_active sau khi trừ W1 vẫn >= min_active_workers)
    W1->>W1: status=draining, ngừng nhận message mới, xử lý nốt job dở

    W2->>Plan: Xin phép chuyển sang draining
    Plan-->>W2: OK (workers_active vẫn còn W3 >= min_active_workers)
    W2->>W2: status=draining, ngừng nhận message mới

    W3->>Plan: Xin phép chuyển sang draining
    Plan-->>W3: Từ chối tạm thời (nếu W3 drain ngay, workers_active=0 < min_active_workers)
    W3->>W3: Vẫn giữ status=active, tiếp tục poll và xử lý message mới từ Queue

    Note over W1,W2: W1 và W2 hoàn tất drain lần lượt, mỗi worker khi restart xong báo lại Plan để cập nhật workers_active
    W1->>Plan: Đã restart xong, workers_active += 1
    Plan->>W3: Cho phép W3 bắt đầu drain (đã có worker khác thay thế)
    W3->>W3: status=draining, ngừng nhận message mới, xử lý nốt job dở rồi restart
```
