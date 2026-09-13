# Enhance sequence — Nâng trust thiết bị sau cửa sổ maturity, cảnh báo qua kênh độc lập

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Một job định kỳ kiểm tra các thiết bị đã hoạt động ổn định qua đủ thời gian maturity window để nâng trust lên `high_value_transfer`, ghi audit trail bất biến và gửi cảnh báo ngay qua kênh độc lập (SMS/email) cho chủ tài khoản. Đáp ứng yêu cầu 1 (cửa sổ maturity) và yêu cầu 5 (audit trail + cảnh báo độc lập) của đề bài.

```mermaid
sequenceDiagram
    participant Job as Device Maturity Job (định kỳ)
    participant DB as Database
    participant Log as TRUST_LEVEL_CHANGE_LOG
    participant Alert as Independent Alert Service
    actor User

    Job->>DB: SELECT DEVICE WHERE trust_level=basic_view AND maturity_window_ends_at <= now()
    DB-->>Job: Danh sách thiết bị đủ điều kiện xét nâng trust

    loop Với mỗi thiết bị đủ điều kiện
        Job->>DB: Kiểm tra thiết bị có hoạt động ổn định trong suốt cửa sổ (không đổi fingerprint bất thường, không có báo cáo gian lận)
        alt Hoạt động ổn định, không có tín hiệu bất thường
            Job->>DB: UPDATE DEVICE SET trust_level=high_value_transfer, trust_upgraded_at=now()
            Job->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(previous=basic_view, new=high_value_transfer, reason=maturity_window_passed)
            Job->>Alert: Gửi cảnh báo độc lập cho chủ tài khoản
            Alert->>Alert: Gửi SMS/email "Thiết bị {tên} vừa được cấp quyền chuyển tiền giá trị lớn, không phải bạn hãy báo ngay"
            Alert-->>User: Nhận cảnh báo qua kênh độc lập, không phụ thuộc vào app đang đăng nhập
        else Có tín hiệu bất thường trong cửa sổ maturity
            Job->>DB: Reset maturity_window_ends_at, giữ nguyên trust_level=basic_view, chờ thêm 1 chu kỳ
            Job->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(reason=maturity_window_passed nhưng bị từ chối do bất thường, new_trust_level giữ nguyên basic_view)
        end
    end
```
