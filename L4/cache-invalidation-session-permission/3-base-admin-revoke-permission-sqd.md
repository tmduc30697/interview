# Base sequence — Admin revoke permission (chỉ invalidate cục bộ)

Đây là **base**, flow "Admin revoke quyền của user" ở trạng thái hiện tại — chỉ invalidate cache ở đúng instance xử lý request revoke, các instance khác không hay biết gì. Flow này liên quan mật thiết tới enhance vì toàn bộ 5 yêu cầu của đề bài đều nhằm sửa đúng lỗ hổng bảo mật này.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminInstance as Service Instance xử lý request admin
    participant DB as USER_PERMISSION store
    participant LocalCache as PERMISSION_CACHE_ENTRY (cục bộ instance này)
    participant OtherInstance as Instance khác (vd Instance B, đang phục vụ user)

    Admin->>AdminInstance: Revoke quyền admin của user X
    AdminInstance->>DB: Cập nhật USER_PERMISSION (xoá quyền)
    DB-->>AdminInstance: Cập nhật thành công
    AdminInstance->>LocalCache: Invalidate PERMISSION_CACHE_ENTRY(user_id=X) — chỉ ở instance này
    AdminInstance-->>Admin: "Revoke thành công"
    Note over OtherInstance: Instance B hoàn toàn không nhận được thông tin gì về việc revoke, cache cục bộ của B vẫn còn quyền cũ
    Note over DB,OtherInstance: User X vẫn dùng được quyền admin trên Instance B cho tới khi ttl_seconds=300 tự hết hạn ở đó, không có log nào ghi lại việc invalidate này
```
