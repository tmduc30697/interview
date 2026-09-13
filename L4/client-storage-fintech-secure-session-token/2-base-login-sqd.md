# Base sequence — Login (cả 2 token đều lưu localStorage)

Đây là **base**, flow đăng nhập ở trạng thái trước enhance. Sau khi xác thực, server trả về cả access token lẫn refresh token, và client lưu cả 2 vào localStorage để dùng cho các lần gọi API sau. Đây chính là điểm mà enhance sẽ thay đổi triệt để: bất kỳ đoạn script nào chạy trên trang, kể cả script bên thứ ba lỡ bị compromise, cũng đọc được các token này.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as Banking Web App
    participant API as Auth API
    participant LS as localStorage (CLIENT_LOCAL_SESSION)

    User->>App: Nhập email/mật khẩu, đăng nhập
    App->>API: POST credentials
    API-->>App: Trả về access_token và refresh_token
    App->>LS: Lưu access_token và refresh_token vào localStorage
    Note over LS: Bất kỳ script nào chạy trên trang đều đọc được cả 2 token này
    App-->>User: Chuyển vào dashboard, dùng access_token từ localStorage cho các API call
```
