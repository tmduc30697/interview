# Base sequence — Gửi SMS (mỗi service tự gọi thẳng gateway)

Đây là **base**, flow "Gửi SMS" ở trạng thái hiện tại — mỗi service nội bộ tự gọi trực tiếp API của đối tác SMS gateway, không qua bất kỳ điểm kiểm soát chung nào. Flow này liên quan mật thiết tới enhance vì toàn bộ vấn đề của đề bài (vượt giới hạn tổng, không có ưu tiên, retry dồn dập, không giám sát) đều bắt nguồn từ việc thiếu một điểm throttle tập trung ở đây.

```mermaid
sequenceDiagram
    actor User as Người dùng đăng nhập
    participant OTP as OTP Service
    participant Mkt as Marketing Service
    participant Gateway as SMS Gateway (đối tác)

    par Cao điểm, nhiều service gửi cùng lúc
        User->>OTP: Yêu cầu OTP đăng nhập
        OTP->>Gateway: Gọi API gửi SMS trực tiếp
    and
        Mkt->>Gateway: Gọi API gửi SMS trực tiếp (campaign marketing)
    end

    Note over OTP,Mkt: Mỗi service tự throttle độc lập theo giả định "được chia đều", không biết tổng lưu lượng thực tế của cả công ty

    Gateway-->>OTP: Rate limit exceeded (429)
    Gateway-->>Mkt: Thành công

    Note over Gateway,OTP: OTP đăng nhập cần độ trễ thấp lại bị chặn ngang hàng với marketing, không có cơ chế ưu tiên nào phân biệt hai loại tin nhắn

    OTP-->>User: Gửi OTP thất bại
```
