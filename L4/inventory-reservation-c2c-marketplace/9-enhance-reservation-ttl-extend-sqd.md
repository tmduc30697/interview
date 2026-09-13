# Sequence - Enhance - Flow "reservation-ttl-extend"

Đây là **enhance**, flow hoàn toàn mới so với base, đáp ứng yêu cầu 4: reservation ở marketplace C2C có TTL ngắn hơn e-commerce chính thức (vì buyer/seller cần thời gian chat để thống nhất giao dịch), và có cơ chế gia hạn khi 2 bên vẫn đang trao đổi thay vì để reservation tự hết hạn giữa chừng.

```mermaid
sequenceDiagram
    actor Buyer
    actor Seller
    participant Chat as Kênh chat
    participant App as Marketplace App
    participant DB as Database
    participant Scheduler as TTL Scheduler

    Note over DB: reservation tạo lúc 10:00:00, TTL C2C ngắn = 15 phút,\nexpires_at = 10:15:00 (ngắn hơn nhiều so với TTL e-commerce)

    Buyer->>Chat: nhắn tin hỏi seller về giao dịch
    Seller->>Chat: phản hồi, 2 bên đang thống nhất giá/giao hàng

    Scheduler-->>Scheduler: tới gần expires_at (còn 2 phút), phát hiện reservation sắp hết hạn

    alt Buyer hoặc Seller còn hoạt động trong kênh chat
        App->>DB: gia hạn expires_at thêm 1 khoảng TTL, extended_count += 1
        App-->>Buyer: reservation được gia hạn, tiếp tục giữ chỗ
    else Không có hoạt động chat nào trong TTL
        Scheduler->>DB: UPDATE reservation SET status = expired WHERE id = X AND status = active
        App->>DB: reserved_count -= 1, giải phóng lại số lượng khả dụng
        App-->>Buyer: thông báo reservation đã hết hạn, cần đặt lại nếu vẫn muốn mua
    end

    Note over DB: extended_count có giới hạn tối đa để tránh giữ chỗ vô thời hạn\nkhi 2 bên chat nhưng không chốt mua
```
