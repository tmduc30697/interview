# Base sequence — Xem tài liệu (quy tắc cách ly cứng nhắc, một loại tài nguyên duy nhất)

Đây là **base**, flow "Xem tài liệu" ở trạng thái hiện tại — chỉ có đúng 1 quy tắc: `document.department_id` phải khớp `employee.department_id` hiện tại, không có ngoại lệ cho tài nguyên dùng chung hay chia sẻ liên phòng ban. Flow này liên quan mật thiết tới enhance vì yêu cầu 1 của đề bài chính là thay quy tắc cứng nhắc này bằng 3 loại tài nguyên với cơ chế kiểm tra khác nhau.

```mermaid
sequenceDiagram
    actor Emp as Nhân viên (phòng kỹ thuật)
    participant App as Internal Tool Service
    participant DB as DOCUMENT store

    Emp->>App: Yêu cầu xem tài liệu D
    App->>DB: Lấy DOCUMENT(D).department_id
    DB-->>App: department_id = "kỹ thuật"
    App->>App: So sánh với employee.department_id hiện tại
    alt Khớp phòng ban
        App-->>Emp: Trả về nội dung tài liệu
    else Không khớp phòng ban (vd tài liệu của phòng tài chính)
        App-->>Emp: Từ chối truy cập, không phân biệt tài liệu đó có phải dùng chung toàn công ty hay không
    end
```
