# Base sequence — Transfer money (không có xác thực bổ sung dù số tiền lớn hay thêm beneficiary mới)

Đây là **base**, flow chuyển tiền hiện tại: giao dịch được thực hiện ngay sau khi user xác nhận trong app, không phân biệt số tiền lớn hay nhỏ, không phân biệt chuyển cho beneficiary đã lưu hay beneficiary mới thêm. Đây là tiền đề cho yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor U as User (session đang hoạt động)
    participant Server
    participant DB as Database

    U->>Server: Thêm beneficiary mới (tên, số tài khoản)
    Server->>DB: INSERT BENEFICIARY
    DB-->>Server: OK
    Server-->>U: Đã thêm beneficiary

    U->>Server: Chuyển 500,000,000 VND cho beneficiary vừa thêm
    Server->>DB: SELECT ACCOUNT balance
    DB-->>Server: đủ số dư
    Server->>DB: INSERT TRANSACTION, UPDATE ACCOUNT balance
    DB-->>Server: OK
    Server-->>U: Chuyển tiền thành công ngay lập tức

    Note over Server,DB: Không có bước xác thực bổ sung nào dù số tiền rất lớn và beneficiary vừa mới thêm, chỉ cần session đăng nhập còn hạn
```
