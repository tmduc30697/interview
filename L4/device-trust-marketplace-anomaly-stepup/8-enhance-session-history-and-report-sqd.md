# Enhance sequence — Xem lịch sử session, báo cáo session lạ, tự động khoá giao dịch treo

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Người dùng xem được danh sách session gần đây (thiết bị, vị trí ước lượng, thời gian) và có thể báo cáo session không phải của mình, hệ thống sẽ tự động thu hồi session đó và khoá tạm các giao dịch đang treo liên quan. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant DB as Database
    participant Report as Session Report Handler

    User->>App: Mở màn hình "Thiết bị và phiên đăng nhập"
    App->>DB: SELECT SESSION WHERE user_id=... ORDER BY last_seen_at DESC
    DB-->>App: Danh sách session (device_name, estimated_city, estimated_country, created_at, last_seen_at, status)
    App-->>User: Hiển thị danh sách kèm nút "Đây không phải tôi" cho từng session

    User->>App: Chọn báo cáo 1 session lạ (vd session tại thành phố người dùng chưa từng tới)
    App->>Report: Tạo SESSION_REPORT (session_id, reported_by_user_id, reported_at)

    Report->>DB: UPDATE SESSION SET status=revoked, flagged_reason=reported_by_user
    Report->>DB: SELECT ORDER WHERE session liên quan AND status=pending

    alt Có giao dịch đang treo được tạo bởi session bị báo cáo
        Report->>DB: UPDATE ORDER SET locked_for_review=true, locked_reason=session_reported_by_owner
        Report-->>User: "Đã khoá tạm N giao dịch đang treo liên quan, đội hỗ trợ sẽ liên hệ xác minh"
    else Không có giao dịch nào đang treo
        Report-->>User: "Đã thu hồi session, không có giao dịch nào bị ảnh hưởng"
    end

    Report->>DB: Ghi SESSION_REPORT.action_taken=session_revoked_and_orders_locked
    App-->>User: Session bị báo cáo không còn dùng được nữa ngay lập tức
```
