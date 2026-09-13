# Base sequence — Reorder tasks drag-drop (mỗi lần kéo thả ghi đè trực tiếp order_index)

Đây là **base**, flow kéo thả sắp xếp lại task trên Gantt chart: mỗi lần kéo thả gửi thẳng `order_index` mới xuống DB, không có version hay coalescing gì cả. Đây là tiền đề cho yêu cầu 3 của đề bài — khi 2 thành viên cùng kéo thả các task khác nhau nhưng cùng ảnh hưởng tới thứ tự chung, việc ghi đè trực tiếp liên tục dễ làm thứ tự cuối cùng bị sai lệch ngoài ý muốn.

```mermaid
sequenceDiagram
    actor C as Thành viên C
    actor D as Thành viên D
    participant Server
    participant DB as Database

    Note over C,D: Cả 2 đang mở cùng Gantt chart, thứ tự hiện tại [T1, T2, T3]

    C->>Server: Kéo T3 lên trước T1 (order_index: T3=1, T1=2, T2=3)
    Server->>DB: UPDATE order_index cho T3, T1, T2
    DB-->>Server: OK

    D->>Server: Kéo T2 lên đầu, dựa trên thứ tự cũ D thấy [T1, T2, T3]
    Note over Server,DB: Server không biết C vừa đổi thứ tự, không kiểm tra gì thêm
    Server->>DB: UPDATE order_index cho T2=1, T1=2, T3=3

    DB-->>Server: OK
    Note over DB: Kết quả cuối [T2, T1, T3] - thay đổi vị trí T3 lên đầu của C bị mất hoàn toàn, không ai được báo conflict
```
