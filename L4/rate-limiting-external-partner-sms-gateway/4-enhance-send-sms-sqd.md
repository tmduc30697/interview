# Enhance sequence — Gửi SMS (qua throttle gateway tập trung, có ưu tiên và TTL)

Đây là **enhance** của flow "Gửi SMS" đã có ở base. So với base, mọi service không còn gọi thẳng đối tác — toàn bộ request phải đi qua một Outbound Throttle Gateway dùng chung, kiểm tra usage window global trước khi quyết định gửi ngay hay xếp hàng theo priority, và request trong hàng đợi có TTL riêng theo loại tin nhắn (đáp ứng yêu cầu 1, 2 và 4 của đề bài).

```mermaid
sequenceDiagram
    actor User as Người dùng đăng nhập
    participant OTP as OTP Service
    participant Mkt as Marketing Service
    participant Throttle as Outbound Throttle Gateway
    participant Window as PARTNER_USAGE_WINDOW
    participant Queue as OUTBOUND_THROTTLE_QUEUE
    participant Gateway as SMS Gateway (đối tác)

    par Cao điểm, nhiều service gửi cùng lúc
        User->>OTP: Yêu cầu OTP đăng nhập
        OTP->>Throttle: Gửi SMS_REQUEST(priority=high, expires_at=+30s)
    and
        Mkt->>Throttle: Gửi SMS_REQUEST(priority=low, expires_at=+5m)
    end

    Throttle->>Window: Kiểm tra request_count hiện tại so với max_requests_per_second (global)

    alt Còn quota trong window hiện tại
        Throttle->>Gateway: Gọi API gửi SMS
        Gateway-->>Throttle: Thành công
        Throttle->>Window: Tăng request_count
        Throttle-->>OTP: Đã gửi
    else Vượt quota, cần xếp hàng
        Throttle->>Queue: Thêm vào OUTBOUND_THROTTLE_QUEUE theo priority
        Note over Queue: Request priority=high (OTP) được xử lý trước request priority=low (marketing) khi quota mở lại
        loop Cho tới khi có quota hoặc hết TTL
            Queue->>Window: Poll quota còn trống
        end
        alt Có quota trước khi expires_at
            Queue->>Gateway: Gọi API gửi SMS
            Gateway-->>Queue: Thành công
            Queue-->>OTP: Đã gửi (trễ)
        else Hết TTL, expires_at đã qua
            Queue->>Queue: Đánh dấu SMS_REQUEST(status=expired)
            Queue-->>OTP: Báo lỗi hết hạn, không gửi
            OTP-->>User: Yêu cầu thử lại (OTP không tới)
        end
    end
```
