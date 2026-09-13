# Enhance sequence — Reorder tasks drag-drop (version riêng cho toàn bộ thứ tự danh sách)

Đây là **enhance** của flow `reorder-tasks-drag-drop` đã có ở base. So với base (mỗi lần kéo thả ghi đè `order_index` trực tiếp, không kiểm tra gì), nay toàn bộ thao tác kéo thả trong 1 phiên được coalesce thành 1 request cập nhật `ordered_task_ids` duy nhất, kiểm tra bằng `order_version` riêng biệt với `TASK.version` — không dùng optimistic lock ở cấp từng task cho việc sắp xếp. Đáp ứng yêu cầu 3 của đề bài.

```mermaid
sequenceDiagram
    actor C as Thành viên C
    actor D as Thành viên D
    participant Client as Client (debounce kéo thả)
    participant Server
    participant DB as Database

    Note over C,D: order_version hiện tại = 7, thứ tự [T1, T2, T3]

    C->>Client: Kéo T3 lên trước T1 (nhiều sự kiện drag liên tiếp)
    Client->>Client: Coalesce các sự kiện drag thành 1 thứ tự cuối cùng
    Client->>Server: PUT project order (order=[T3,T1,T2], order_version=7)
    Server->>DB: UPDATE PROJECT_TASK_ORDER SET ordered_task_ids=[T3,T1,T2], order_version=8 WHERE order_version=7
    DB-->>Server: 1 row affected, OK
    Server-->>Client: Lưu thành công, order_version=8

    D->>Client: Kéo T2 lên đầu, dựa trên order_version=7 D đã tải trước đó
    Client->>Server: PUT project order (order=[T2,T1,T3], order_version=7)
    Server->>DB: UPDATE PROJECT_TASK_ORDER ... WHERE order_version=7
    DB-->>Server: 0 row affected, order_version hiện tại đã là 8
    Server-->>Client: Conflict, trả về thứ tự mới nhất [T3,T1,T2] và order_version=8
    Client-->>D: Hiển thị "thứ tự đã được cập nhật bởi người khác", D kéo lại trên thứ tự mới

    Note over Server,DB: Vì order_version tách riêng khỏi TASK.version, việc kéo-thả không làm tăng version của từng task, tránh conflict giả khi A/B ở flow update-task đang sửa field khác của task
```
