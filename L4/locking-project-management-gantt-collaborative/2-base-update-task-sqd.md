# Base sequence — Update task (không có kiểm soát tương tranh, lost update)

Đây là **base**, flow chỉnh sửa task hiện tại: server ghi đè trực tiếp theo dữ liệu request gửi lên, không kiểm tra task đã bị người khác sửa từ lúc mình đọc hay chưa. Đây chính là tiền đề cho vấn đề nêu ở yêu cầu 1 và 2 của đề bài — thành viên A đổi deadline và thành viên B gán thêm người phụ trách cho cùng task X, ai lưu sau sẽ ghi đè mất thay đổi hợp lệ của người lưu trước.

```mermaid
sequenceDiagram
    actor A as Thành viên A
    actor B as Thành viên B
    participant Server
    participant DB as Database

    A->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (deadline=10, assignee=[])
    Server-->>A: task X

    B->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (deadline=10, assignee=[])
    Server-->>B: task X

    A->>Server: PUT task X (deadline=15)
    Server->>DB: UPDATE task X SET deadline=15
    DB-->>Server: OK
    Server-->>A: Lưu thành công

    B->>Server: PUT task X (assignee=[U2]) dựa trên dữ liệu cũ đã đọc
    Note over Server,DB: Server không kiểm tra task đã đổi kể từ lúc B đọc
    Server->>DB: UPDATE task X SET assignee=[U2], deadline=10 (ghi đè theo payload cũ của B)
    DB-->>Server: OK
    Server-->>B: Lưu thành công

    Note over DB: Deadline=15 của A bị ghi đè về 10, lost update không ai biết
```
