# Enhance sequence — Step-up khẩn cấp qua kênh độc lập cho giao dịch gấp hợp lệ

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Khi giao dịch bị chặn vì thiết bị chưa đủ trust nhưng người dùng hợp lệ thực sự cần gấp, họ có thể yêu cầu xác minh bổ sung qua kênh độc lập (gọi điện, video call, sinh trắc học bổ sung) để nâng trust ngay lập tức thay vì chờ hết maturity window. Đáp ứng yêu cầu 4, đồng thời vẫn đảm bảo audit trail và cảnh báo độc lập theo yêu cầu 5.

```mermaid
sequenceDiagram
    actor User as Chủ tài khoản hợp lệ
    participant App as Mobile App
    participant Bank as Transfer Service
    participant Verify as Step-up Verification Service
    participant DB as Database
    participant Log as TRUST_LEVEL_CHANGE_LOG
    participant Alert as Independent Alert Service

    User->>App: Giao dịch bị từ chối do device_trust_insufficient
    App-->>User: Hiển thị lựa chọn "Xác minh khẩn cấp ngay" (gọi điện/video call/sinh trắc học bổ sung)
    User->>App: Chọn xác minh khẩn cấp

    App->>Verify: Tạo STEPUP_VERIFICATION (device_id, transaction_id, method=video_call, status=pending)
    Verify->>User: Kết nối video call với tổng đài viên hoặc yêu cầu sinh trắc học bổ sung ngay trên app
    User->>Verify: Hoàn tất xác minh danh tính (khớp giấy tờ/khuôn mặt/giọng nói)

    alt Xác minh thành công
        Verify->>DB: UPDATE STEPUP_VERIFICATION SET status=verified, verified_at=now()
        Verify->>DB: UPDATE DEVICE SET trust_level=high_value_transfer, trust_upgraded_at=now()
        Verify->>Log: Ghi TRUST_LEVEL_CHANGE_LOG(previous=basic_view, new=high_value_transfer, reason=stepup_verified)
        Verify->>Alert: Gửi cảnh báo độc lập cho chủ tài khoản
        Alert->>Alert: Gửi SMS/email "Thiết bị {tên} vừa được xác minh khẩn cấp và cấp quyền giao dịch lớn, không phải bạn hãy báo ngay"

        App->>Bank: Thử lại giao dịch bị chặn trước đó
        Bank->>DB: Đọc lại trust_level, nay đã là high_value_transfer
        Bank->>DB: Trừ/cộng số dư, UPDATE TRANSACTION SET status=success
        Bank-->>App: Chuyển tiền thành công
        App-->>User: Giao dịch hoàn tất ngay trong phiên xác minh, không phải chờ hàng giờ/ngày
    else Xác minh thất bại hoặc nghi ngờ gian lận
        Verify->>DB: UPDATE STEPUP_VERIFICATION SET status=failed
        Verify-->>App: Từ chối nâng trust, giao dịch tiếp tục bị chặn
        App-->>User: Hướng dẫn liên hệ tổng đài hỗ trợ trực tiếp để xử lý thêm
    end
```
