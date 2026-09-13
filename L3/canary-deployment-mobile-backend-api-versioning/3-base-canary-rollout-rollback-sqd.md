# Base sequence — Canary rollout & rollback (chưa nhận diện version app)

Đây là **base**, flow "Canary rollout" ở trạng thái hiện tại — traffic canary chọn ngẫu nhiên không phân biệt phiên bản app, metric lỗi gộp chung, rollback chỉ đơn giản là revert code. Flow này liên quan mật thiết tới enhance vì yêu cầu 2, 3, 4 và 5 của đề bài đều nhằm sửa đúng các lỗ hổng ở đây.

```mermaid
sequenceDiagram
    participant Router as Router
    participant Canary as Backend canary
    participant Stable as Backend stable
    participant Metric as ERROR_METRIC_WINDOW store
    actor OnCall as On-call Engineer
    actor NewApp as Client app mới nhất
    actor OldApp as Client app phiên bản cũ

    Router->>Router: Random 5% traffic vào canary, không xét app_client_version_header
    Note over Router: Đa số user đã update app mới nên ngẫu nhiên dễ dồn canary vào nhóm NewApp, ít chạm tới OldApp
    NewApp->>Canary: Request (thường xuyên rơi vào canary)
    OldApp->>Stable: Request (hiếm khi rơi vào canary do ngẫu nhiên thiên lệch)
    Canary->>Metric: Ghi error_rate gộp chung mọi client version
    Metric-->>OnCall: error_rate tổng thể vẫn thấp (vì OldApp ít vào canary nên lỗi parse ở nhóm này chưa lộ ra)
    OnCall->>Router: Thấy ổn, tiếp tục tăng traffic canary
    Note over Canary,OldApp: Khi traffic tăng cao hơn, OldApp mới bắt đầu dính nhiều vào canary và lộ ra lỗi parse response — phát hiện quá muộn
    OnCall->>Canary: Nếu phát hiện lỗi, rollback = deploy lại code backend cũ
    Note over Canary,Stable: Nếu canary đã ghi dữ liệu theo schema mới trước khi rollback, code cũ có thể không đọc được — chưa có kế hoạch xử lý
```
