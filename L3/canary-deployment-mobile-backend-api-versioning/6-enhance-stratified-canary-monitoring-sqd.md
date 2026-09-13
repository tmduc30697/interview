# Enhance sequence — Stratified canary traffic & per-version monitoring

Đây là **enhance**, cùng flow "Canary rollout" đã có ở base nhưng nay thay đổi theo yêu cầu 2, 3 và 4 của đề bài: traffic canary được stratify để bao gồm cả app cũ lẫn mới, và metric lỗi tách riêng theo `app_client_version_header` để phát hiện lỗi parse chỉ ảnh hưởng nhóm client cũ.

```mermaid
sequenceDiagram
    participant Router as Router
    participant Strat as CANARY_TRAFFIC_STRATIFICATION store
    participant Canary as Backend canary
    participant Metric as CLIENT_VERSION_ERROR_METRIC store
    actor NewApp as Client app mới nhất
    actor OldApp as Client app phiên bản cũ

    Router->>Strat: Lấy target_percent_of_canary theo từng app_version
    Note over Strat: Đảm bảo cả nhóm OldApp lẫn NewApp đều có tỷ lệ đủ lớn rơi vào canary, không chỉ dựa random đơn thuần
    NewApp->>Router: Request kèm header app_client_version=4.5
    Router->>Canary: Route theo đúng tỷ lệ stratify cho version 4.5
    OldApp->>Router: Request kèm header app_client_version=3.2
    Router->>Canary: Route theo đúng tỷ lệ stratify cho version 3.2 (đảm bảo có mặt trong canary ngay từ đầu)

    Canary->>Metric: Ghi CLIENT_VERSION_ERROR_METRIC riêng theo app_client_version_header (error_rate, parse_error_rate)

    alt parse_error_rate của nhóm OldApp (v3.2) tăng bất thường
        Metric-->>Router: Cảnh báo riêng cho nhóm v3.2, dù error_rate tổng thể vẫn thấp
        Router->>Router: Kích hoạt ROLLBACK_EVENT(affected_app_versions=v3.2)
        Note over Metric: Nhờ tách theo version, lỗi chỉ ảnh hưởng thiểu số client cũ không bị chìm trong tỷ lệ lỗi tổng thể
    else Mọi nhóm version đều ổn định
        Metric-->>Router: Cho phép tiếp tục tăng traffic canary
    end
```
