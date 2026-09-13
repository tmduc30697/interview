# Base sequence — Change member role (không có tác động MFA vì chưa có chính sách)

Đây là **base**, flow admin đổi role của 1 thành viên trong tổ chức. Ở base, việc đổi role chỉ ảnh hưởng tới quyền truy cập tính năng (permission), hoàn toàn không liên quan gì tới MFA vì tổ chức chưa có chính sách MFA theo role. Đây là tiền đề cho yêu cầu 2 của đề bài.

```mermaid
sequenceDiagram
    actor Admin
    actor Viewer as User (đang là viewer)
    participant Server
    participant DB as Database

    Admin->>Server: PUT org member role (user=Viewer, role=admin)
    Server->>DB: UPDATE ORG_MEMBERSHIP SET role=admin WHERE user_id=Viewer, org_id=Org1
    DB-->>Server: OK
    Server-->>Admin: Đổi role thành công

    Note over Viewer: Session hiện tại của Viewer vẫn tiếp tục hoạt động bình thường, không có kiểm tra gì thêm liên quan MFA vì tổ chức chưa có chính sách nào để áp dụng
```
