# Enhance sequence — Transfer money (step-up MFA khi vượt ngưỡng hoặc thêm beneficiary mới)

Đây là **enhance** của flow `transfer-money` đã có ở base. So với base (chuyển tiền ngay không cần xác thực thêm), nay giao dịch vượt ngưỡng số tiền hoặc chuyển cho beneficiary mới thêm phải yêu cầu xác thực MFA lại, dù session đăng nhập còn hạn. Đáp ứng yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (session đang hoạt động, đã login MFA trước đó)
    participant Server
    participant DB as Database

    U->>Server: Thêm beneficiary mới (tên, số tài khoản)
    Server->>DB: INSERT BENEFICIARY
    DB-->>Server: OK
    Server-->>U: Đã thêm beneficiary

    U->>Server: Chuyển 500,000,000 VND cho beneficiary vừa thêm
    Server->>DB: SELECT STEP_UP_POLICY
    DB-->>Server: amount_threshold=50,000,000, require_for_new_beneficiary=true
    Server->>Server: Kiểm tra amount (500tr > ngưỡng) VÀ beneficiary mới thêm trong phiên này
    Note over Server: Cả 2 điều kiện đều kích hoạt step-up, dù session vẫn còn hạn

    Server->>DB: INSERT TRANSACTION (status=pending, required_step_up=true)
    Server-->>U: Yêu cầu xác thực MFA lại để xác nhận giao dịch này

    U->>Server: Nhập mã OTP từ TOTP app
    Server->>DB: Xác thực OTP hợp lệ
    Server->>DB: UPDATE TRANSACTION SET step_up_verified_at=now(), status=completed
    Server->>DB: UPDATE ACCOUNT balance
    DB-->>Server: OK

    Server-->>U: Chuyển tiền thành công

    Note over Server,DB: Nếu giao dịch dưới ngưỡng và gửi cho beneficiary đã lưu từ trước, không cần step-up, chỉ session hợp lệ là đủ
```
