# Enhance sequence — Transfer money (kiểm tra trust thiết bị, mặc định từ chối khi chưa đủ, chặn race condition)

Đây là **enhance** của flow `transfer-money` đã có ở base. So với base (chuyển tiền không kiểm tra gì), nay mọi giao dịch vượt ngưỡng giá trị lớn bắt buộc kiểm tra `DEVICE.trust_level=high_value_transfer` ngay tại thời điểm xử lý giao dịch (không dựa vào trạng thái đã cache lúc login), mặc định từ chối nếu thiết bị chưa đạt mức trust này — kể cả khi vừa đăng nhập xong vài giây trước. Đáp ứng yêu cầu 2 của đề bài, tái hiện đúng kịch bản tấn công ở base nhưng lần này bị chặn.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (thiết bị mới, vừa đăng nhập)
    participant App as Mobile App
    participant Bank as Transfer Service
    participant DB as Database

    Attacker->>App: Đăng nhập thành công trên thiết bị mới (trust_level=basic_view, maturity_window chưa qua)
    Note over Attacker,App: Chỉ vài giây sau khi đăng nhập

    Attacker->>App: Yêu cầu chuyển khoản giá trị lớn ra tài khoản lạ
    App->>Bank: Thực hiện TRANSACTION (type=transfer, amount=lớn)
    Bank->>DB: Đọc trust_level hiện tại của DEVICE ngay tại thời điểm xử lý (không dùng giá trị đã cache lúc login)
    DB-->>Bank: trust_level=basic_view (chưa qua maturity window, chưa có stepup)

    alt amount vượt ngưỡng giá trị lớn AND trust_level != high_value_transfer
        Bank->>DB: UPDATE TRANSACTION SET status=blocked_awaiting_stepup, blocked_reason=device_trust_insufficient
        Bank-->>App: Từ chối giao dịch, mặc định deny
        App-->>Attacker: "Thiết bị chưa đủ tin cậy để chuyển khoản giá trị lớn, cần xác minh bổ sung"
        Note over Bank,DB: Mặc định từ chối ngay cả khi thiết bị vừa đăng nhập thành công bằng đúng mật khẩu/MFA, loại bỏ race condition giữa lúc thiết lập trust và lúc thực hiện giao dịch
    else trust_level=high_value_transfer đã được xác lập từ trước
        Bank->>DB: Trừ/cộng số dư, UPDATE TRANSACTION SET status=success
        Bank-->>App: Chuyển tiền thành công
    end
```
