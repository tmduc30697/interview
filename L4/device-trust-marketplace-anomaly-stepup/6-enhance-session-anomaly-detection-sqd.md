# Enhance sequence — Phát hiện bất thường trong phiên, gắn cờ nghi ngờ mà không đăng xuất ngay

Đây là **enhance**, flow hoàn toàn mới, chưa tồn tại ở base. Mỗi request đều được ghi vào `SESSION_ACTIVITY_LOG` để so sánh IP/vị trí địa lý và user-agent giữa các request liên tiếp, phát hiện dấu hiệu session bị chiếm quyền. Khi bị gắn cờ nghi ngờ, hệ thống chỉ chặn hành động nhạy cảm chứ không đăng xuất người dùng nếu họ chỉ đang xem sản phẩm. Đáp ứng yêu cầu 2 và 3 của đề bài.

```mermaid
sequenceDiagram
    actor User
    participant App as Marketplace App
    participant Monitor as Session Anomaly Monitor
    participant DB as Database

    User->>App: Request 1 - Xem sản phẩm (IP tại Hà Nội, user-agent Chrome/Android)
    App->>DB: Ghi SESSION_ACTIVITY_LOG (ip, estimated_city=Hà Nội, user_agent=Chrome/Android, requested_at)

    Note over User,App: 5 phút sau, cùng session token nhưng request đến từ nơi khác

    User->>App: Request 2 - Xem sản phẩm khác (IP tại một quốc gia khác, user-agent Firefox/Windows)
    App->>DB: Ghi SESSION_ACTIVITY_LOG (ip mới, estimated_city khác, user_agent khác, requested_at)

    App->>Monitor: So sánh activity log request liên tiếp gần nhất của session
    Monitor->>Monitor: Phát hiện đổi vị trí địa lý bất hợp lý trong 5 phút (không thể di chuyển vật lý), đồng thời đổi user-agent giữa chừng

    Monitor->>DB: UPDATE SESSION SET status=flagged_suspicious, flagged_reason=ip_geo_jump

    alt Request tiếp theo chỉ là xem sản phẩm/duyệt danh mục
        App-->>User: Vẫn cho xem bình thường, không đăng xuất, không làm phiền
        Note over App,Monitor: Tránh false positive làm phiền người dùng hợp lệ (vd đổi mạng, dùng VPN) khi họ chưa chạm hành động nhạy cảm nào
    else Request tiếp theo là hành động nhạy cảm (đổi địa chỉ, đổi thanh toán, checkout)
        App->>App: Chặn lại, chuyển sang flow xác minh bổ sung bắt buộc trước khi cho tiếp tục
        App-->>User: "Phát hiện dấu hiệu bất thường, vui lòng xác minh lại trước khi tiếp tục"
    end
```
