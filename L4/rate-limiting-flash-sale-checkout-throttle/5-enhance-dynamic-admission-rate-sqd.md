# Enhance sequence — Điều chỉnh admission rate động theo sức khỏe backend

Đây là **enhance**, flow hoàn toàn mới, phát sinh từ enhance — chưa tồn tại ở base (base không có khái niệm admit rate). Đáp ứng yêu cầu 3 của đề bài: tốc độ cho qua hàng chờ phải phản ánh tải thật của backend, không dùng số cố định tĩnh.

```mermaid
sequenceDiagram
    participant DB as Order/Inventory DB
    participant Health as Backend Health Monitor
    participant Snapshot as BACKEND_HEALTH_SNAPSHOT
    participant Config as ADMISSION_RATE_CONFIG
    participant WR as Waiting Room Service

    loop Định kỳ (ví dụ mỗi vài giây)
        Health->>DB: Đo latency, error rate hiện tại
        Health->>Snapshot: Ghi BACKEND_HEALTH_SNAPSHOT(db_latency_ms, error_rate_pct)

        alt Latency/error rate vượt ngưỡng an toàn
            Snapshot->>Config: Yêu cầu giảm current_admit_rate_per_sec
            Config->>Config: Cập nhật rate mới, reason=db_latency_high
        else Backend đang khỏe, còn dư sức chịu tải
            Snapshot->>Config: Yêu cầu tăng dần current_admit_rate_per_sec (không tăng đột ngột)
            Config->>Config: Cập nhật rate mới, reason=healthy
        end
    end

    WR->>Config: Đọc current_admit_rate_per_sec trước mỗi vòng cấp ADMISSION_TOKEN

    Note over WR,Config: Số lượng user được cấp token mỗi vòng luôn theo rate mới nhất, phản ánh tải thật thay vì con số cấu hình cứng
```
