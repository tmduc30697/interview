# Enhance sequence — Update task đã bị xóa (conflict rõ ràng, không tạo lại nhầm)

Đây là **enhance**, flow mới phát sinh từ đề bài, chưa có ở base (base chưa hỗ trợ xóa task nên chưa có kịch bản này). Khi thành viên E xóa task X (soft delete) đúng lúc thành viên F đang sửa task X dựa trên version cũ trước khi bị xóa, request update phải nhận lỗi rõ ràng "task đã bị xóa" thay vì tạo lại task hoặc báo lỗi version mismatch mơ hồ. Đáp ứng yêu cầu 4 của đề bài.

```mermaid
sequenceDiagram
    actor E as Thành viên E
    actor F as Thành viên F
    participant Server
    participant DB as Database

    F->>Server: GET task X
    Server->>DB: SELECT task X
    DB-->>Server: task X (version=5, deleted_at=null)
    Server-->>F: task X (version=5)

    E->>Server: DELETE task X (version=5)
    Server->>DB: UPDATE task X SET deleted_at=now(), version=6 WHERE version=5
    DB-->>Server: OK
    Server-->>E: Xóa thành công

    F->>Server: PUT task X (deadline=20, version=5)
    Server->>DB: SELECT task X WHERE id=X
    DB-->>Server: task X (version=6, deleted_at=<timestamp>)
    Note over Server: Version không khớp (5 vs 6), kiểm tra nguyên nhân trước khi merge
    Server->>DB: Kiểm tra deleted_at, phát hiện task đã bị xóa
    Server-->>F: Lỗi rõ ràng "Task đã bị xóa bởi thành viên khác", không tạo lại task, không áp update
```
