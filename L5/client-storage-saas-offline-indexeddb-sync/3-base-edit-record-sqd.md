# Base sequence — Edit record (ghi thẳng server, thất bại ngay khi offline)

Đây là **base**, flow chỉnh sửa 1 bản ghi ở trạng thái trước enhance. Mọi thao tác sửa/xoá đều gọi thẳng API, không có bước lưu tạm cục bộ nào. Khi mất mạng, thao tác thất bại ngay lập tức và có thể mất nội dung vừa sửa. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi thao tác offline được ghi vào outbox để replay sau.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant API as Record API
    participant DB as Server DB

    User->>App: Sửa nội dung 1 bản ghi (task/ticket/contact)
    App->>API: PATCH record với nội dung mới
    alt Có mạng
        API->>DB: Cập nhật RECORD, updated_at
        API-->>App: Xác nhận đã lưu
        App-->>User: Hiển thị đã lưu thành công
    else Mất mạng
        API-->>App: Request thất bại (network error)
        App-->>User: Báo lỗi lưu thất bại, không có cơ chế lưu tạm hay tự động thử lại sau
    end
```
