# Enhance sequence — Update percent_complete (atomic increment, bỏ qua optimistic lock)

Đây là **enhance**, flow mới phát sinh từ đề bài cho riêng trường `percent_complete` — trường có tần suất tranh chấp cao hơn hẳn các trường khác vì nhiều thành viên cùng cập nhật liên tục. Thay vì dùng optimistic lock như flow `update-task` (sẽ gây tỷ lệ conflict/retry rất cao cho trường này), hệ thống dùng atomic increment ở tầng DB, bỏ qua hoàn toàn kiểm tra version cho riêng trường này. Đáp ứng yêu cầu 5 của đề bài.

```mermaid
sequenceDiagram
    actor G as Thành viên G
    actor H as Thành viên H
    participant Server
    participant DB as Database

    Note over G,H: task X đang có percent_complete=40, version=9 (version này không bị đụng tới bởi 2 update dưới đây)

    G->>Server: PATCH task X percent_complete += 10
    Server->>DB: UPDATE task X SET percent_complete = LEAST(100, percent_complete + 10) WHERE id=X
    DB-->>Server: OK, percent_complete=50

    H->>Server: PATCH task X percent_complete += 5 (gần như đồng thời với G)
    Server->>DB: UPDATE task X SET percent_complete = LEAST(100, percent_complete + 5) WHERE id=X
    DB-->>Server: OK, percent_complete=55

    Server-->>G: Cập nhật thành công, percent_complete hiện tại=55
    Server-->>H: Cập nhật thành công, percent_complete hiện tại=55

    Note over Server,DB: Không cần đọc version trước, không cần retry khi conflict, cả 2 update đều được cộng dồn đúng vì DB tự đảm bảo tính atomic của phép increment, đổi lại tính năng này không hỗ trợ merge phức tạp như flow update-task
```
