# Base sequence — Transfer money (không kiểm tra trust thiết bị, lộ rủi ro kẻ tấn công)

Đây là **base**, flow chuyển tiền ở trạng thái hiện tại: sau khi đăng nhập thành công, thiết bị được phép chuyển tiền với bất kỳ số tiền nào ngay lập tức, không có bước kiểm tra mức độ tin cậy thiết bị. Sơ đồ minh hoạ đúng kịch bản rủi ro nêu ở yêu cầu 2 của đề bài: kẻ tấn công chiếm được thông tin đăng nhập, đăng nhập trên thiết bị mới rồi chuyển tiền lớn thành công chỉ trong vài giây.

```mermaid
sequenceDiagram
    actor Attacker as Kẻ tấn công (có username/password bị lộ)
    participant App as Mobile App
    participant Auth as Auth Service
    participant Bank as Transfer Service
    participant DB as Database

    Attacker->>App: Đăng nhập trên thiết bị hoàn toàn mới với thông tin đánh cắp
    App->>Auth: Xác thực password + MFA (kẻ tấn công cũng chiếm được OTP qua SIM-swap/phishing)
    Auth-->>App: Xác thực thành công
    App->>DB: Ghi nhận DEVICE mới, không gán mức trust

    Note over Attacker,App: Chỉ vài giây sau khi đăng nhập thành công

    Attacker->>App: Yêu cầu chuyển khoản giá trị lớn ra tài khoản lạ
    App->>Bank: Thực hiện TRANSACTION (type=transfer, amount=lớn)
    Bank->>DB: Không có kiểm tra nào về độ tin cậy thiết bị vừa đăng nhập
    DB-->>Bank: Đủ số dư, không có rào cản nào khác
    Bank-->>App: Chuyển tiền thành công
    App-->>Attacker: Giao dịch hoàn tất, nạn nhân chỉ phát hiện sau khi tiền đã mất
```
