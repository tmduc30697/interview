# Enhance sequence — Update task (optimistic lock cấp task + field-level merge)

Đây là **enhance** của flow `update-task` đã có ở base. So với base (ghi đè trực tiếp, lost update), nay mỗi update phải kèm `version` đã đọc; nếu version không khớp, hệ thống kiểm tra field-level thay vì từ chối toàn bộ ngay — nếu 2 request sửa các trường không giao nhau (A đổi deadline, B thêm assignee) thì merge cả hai vào bản ghi mới. Đáp ứng đúng kịch bản mô tả ở yêu cầu 1 và 2 của đề bài.

```mermaid
sequenceDiagram
    actor A as Thành viên A
    actor B as Thành viên B
    participant Server
    participant DB as Database

    A->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (version=3, deadline=10, assignee=[])
    Server-->>A: task X (version=3)

    B->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (version=3, deadline=10, assignee=[])
    Server-->>B: task X (version=3)

    A->>Server: PUT task X (deadline=15, version=3)
    Server->>DB: UPDATE task X SET deadline=15, version=4 WHERE version=3
    DB-->>Server: 1 row affected, OK
    Server-->>A: Lưu thành công, version=4

    B->>Server: PUT task X (assignee=[U2], version=3)
    Server->>DB: UPDATE task X SET assignee=[U2], version=4 WHERE version=3
    DB-->>Server: 0 row affected, version hiện tại đã là 4
    Note over Server: Version mismatch, kiểm tra field-level thay vì reject ngay
    Server->>DB: Diff các trường B đổi (assignee) so với các trường A đã đổi (deadline)
    Note over Server: Không giao nhau, field-level merge chấp nhận được
    Server->>DB: UPDATE task X SET assignee=[U2], version=5 WHERE version=4
    DB-->>Server: OK
    Server-->>B: Lưu thành công (đã merge), version=5

    Note over Server,DB: Trade-off, field-level merge tăng khả năng song song nhưng rủi ro nếu 2 trường có phụ thuộc chéo với nhau, ví dụ 1 field validate dựa trên field kia, nên chỉ áp dụng merge cho các trường thực sự độc lập, còn lại vẫn reject theo optimistic lock thông thường
```
