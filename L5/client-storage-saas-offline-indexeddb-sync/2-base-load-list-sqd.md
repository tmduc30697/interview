# Base sequence — Load list (luôn gọi API trực tiếp, không có cache)

Đây là **base**, flow tải danh sách bản ghi (task/ticket/contact) ở trạng thái trước enhance. Mỗi lần vào trang, dashboard gọi thẳng API lấy dữ liệu, chờ phản hồi rồi mới render, không có bất kỳ cache cục bộ nào. Nếu mất mạng, người dùng chỉ thấy lỗi hoặc màn hình trống. Flow này là nền để so sánh với flow cùng tên ở enhance, nơi IndexedDB được đọc trước để hiển thị ngay.

```mermaid
sequenceDiagram
    actor User as Người dùng
    participant App as SaaS Dashboard
    participant API as List API
    participant DB as Server DB

    User->>App: Mở trang danh sách task/ticket/contact
    App->>API: GET danh sách theo project
    alt Có mạng, API phản hồi bình thường
        API->>DB: Truy vấn danh sách bản ghi
        DB-->>API: Trả về hàng chục nghìn bản ghi
        API-->>App: Trả kết quả
        App-->>User: Render danh sách sau khi có đủ dữ liệu
    else Mất mạng hoặc API lỗi
        API-->>App: Timeout/lỗi kết nối
        App-->>User: Hiển thị lỗi hoặc màn hình trống, không có dữ liệu nào để xem tạm
    end
```
