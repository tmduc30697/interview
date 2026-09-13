# Base sequence — Authorize request

Đây là **base**, flow "Kiểm tra quyền cho mỗi request" — nền tảng cho enhance: mỗi instance tự cache permission cục bộ, không biết gì về thay đổi quyền xảy ra ở nơi khác cho tới khi TTL của chính nó hết hạn.

```mermaid
sequenceDiagram
    actor User
    participant Instance as Service Instance (vd Instance A)
    participant Cache as PERMISSION_CACHE_ENTRY store (cục bộ Instance A)
    participant DB as USER_PERMISSION store

    User->>Instance: Gửi request tới tài nguyên cần quyền
    Instance->>Cache: Đọc PERMISSION_CACHE_ENTRY(user_id, instance_id=A)
    alt Cache hit
        Cache-->>Instance: Trả permissions_snapshot cục bộ
    else Cache miss
        Cache-->>Instance: Miss
        Instance->>DB: Đọc USER_PERMISSION từ DB
        DB-->>Instance: Trả permission hiện tại
        Instance->>Cache: Ghi cache cục bộ, ttl_seconds=300
    end
    Instance->>Instance: Kiểm tra quyền theo permissions_snapshot vừa có
    Instance-->>User: Cho phép/từ chối theo quyền trong cache
```
