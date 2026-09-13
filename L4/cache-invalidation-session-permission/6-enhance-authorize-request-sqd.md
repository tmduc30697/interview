# Enhance sequence — Authorize request (test case: revoke giữa session đang hoạt động)

Đây là **enhance**, cùng flow "Authorize request" đã có ở base nhưng nay thay đổi theo yêu cầu 3 và 4 của đề bài: TTL dự phòng rất ngắn thay vì vài phút, và minh hoạ đúng test case bắt buộc — user đang có session hoạt động bị revoke quyền admin giữa chừng, request tiếp theo phải bị chặn đúng theo quyền mới.

```mermaid
sequenceDiagram
    actor User
    participant Instance as Service Instance A (đang phục vụ user)
    participant Cache as PERMISSION_CACHE_ENTRY store (cục bộ Instance A)
    actor Admin
    participant PubSub as Pub/Sub broadcast

    User->>Instance: Request 1 — đang dùng quyền admin, cache hit, cho phép
    Instance-->>User: Cho phép (quyền admin còn hiệu lực trong cache)

    Admin->>PubSub: (song song) Revoke quyền admin của user này — xem flow "Admin revoke permission"
    PubSub->>Instance: Broadcast PERMISSION_INVALIDATION_EVENT tới Instance A
    Instance->>Cache: Invalidate PERMISSION_CACHE_ENTRY(user_id) ngay khi nhận event

    User->>Instance: Request 2 — gửi ngay sau đó, vẫn dùng session cũ
    Instance->>Cache: Đọc PERMISSION_CACHE_ENTRY(user_id)
    alt Đã kịp nhận broadcast invalidation
        Cache-->>Instance: Miss (đã bị invalidate)
        Instance->>Instance: Đọc lại permission mới nhất từ DB, ghi cache mới với ttl_seconds=5
        Instance-->>User: Từ chối — quyền admin đã bị revoke
    else Broadcast bị lỡ (Instance A mất kết nối tạm thời với PubSub)
        Cache-->>Instance: Vẫn còn cache cũ, nhưng ttl_seconds chỉ 5 giây nên sắp hết hạn
        Instance-->>User: Vẫn cho phép trong tối đa vài giây (cửa sổ rủi ro giới hạn), sau đó cache tự hết hạn và request kế tiếp bị chặn đúng
    end
```
