# Cache session/permission cho hệ thống phân quyền

**Hệ thống:** Một SaaS B2B cache thông tin quyền hạn (permission/role) của user để tránh query DB mỗi request, ảnh hưởng tới việc kiểm soát truy cập.

**Vai trò của flow:** Invalidate cache permission ngay khi quyền của user bị thay đổi (bị revoke quyền, đổi role), vì đây là dữ liệu bảo mật — cache cũ có thể dẫn tới lỗ hổng cho phép truy cập trái phép.

**Yêu cầu cụ thể:**
- Khi admin revoke quyền của một user, cache permission liên quan phải bị invalidate ngay lập tức và có xác nhận (không chấp nhận eventual consistency dạng "vài giây sau mới có hiệu lực" cho việc revoke quyền).
- Trong kiến trúc nhiều instance service cùng cache permission cục bộ, phải có cơ chế broadcast invalidation tới toàn bộ instance (ví dụ qua pub/sub) khi có thay đổi quyền, không chỉ invalidate ở một instance xử lý request đổi quyền.
- Nếu broadcast invalidation thất bại một phần (một instance không nhận được message do mất kết nối tạm thời), phải có cơ chế dự phòng (TTL rất ngắn cho cache permission, ví dụ vài giây) để giới hạn "cửa sổ rủi ro" tối đa.
- Có test case cụ thể: user đang có session hoạt động bị revoke quyền admin ngay khi đang dùng — request tiếp theo của họ (sau invalidation) phải bị chặn đúng theo quyền mới, không dùng permission cache cũ.
- Log đầy đủ mọi lần invalidate permission cache (ai, quyền gì, khi nào) để phục vụ audit an ninh khi cần điều tra truy cập trái phép.

---

# Base ERD — Cache permission trước khi có broadcast invalidation

Đây là **base**: mô hình dữ liệu suy luận từ ngữ cảnh đề bài, thể hiện trạng thái hệ thống **trước khi** có cơ chế broadcast invalidation xuyên instance. Đề bài nói kiến trúc nhiều instance cùng cache permission cục bộ — nên base cần đủ: user, role/permission gán cho user, và cache entry permission theo từng instance với TTL cố định. Chưa có entity nào phục vụ broadcast/xác nhận/audit log invalidation — những thứ đó là phần enhance.

```mermaid
erDiagram
    USER ||--o{ USER_PERMISSION : has
    USER ||--o{ PERMISSION_CACHE_ENTRY : "cached per instance"

    USER {
        string id PK
        string name
    }
    USER_PERMISSION {
        string id PK
        string user_id FK
        string permission
        string role
    }
    PERMISSION_CACHE_ENTRY {
        string cache_key PK "user_id + instance_id"
        string instance_id
        string permissions_snapshot
        datetime cached_at
        int ttl_seconds "vd 300"
    }
```

---

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

---

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

---

# Enhance ERD — sau khi có broadcast invalidation xuyên instance

Đây là ERD **sau khi** enhance được áp dụng lên base. So với base, có 3 nhóm entity mới, ứng trực tiếp với 5 yêu cầu trong đề bài:

- `PERMISSION_INVALIDATION_EVENT` (mới) — phát broadcast ngay khi có revoke, thay vì chỉ invalidate cục bộ như base.
- `PERMISSION_INVALIDATION_ACK` (mới) — theo dõi từng instance đã xác nhận invalidate hay chưa, đảm bảo lan tới toàn bộ instance.
- `PERMISSION_CACHE_ENTRY` giảm `ttl_seconds` xuống rất ngắn (dự phòng cho trường hợp broadcast thất bại một phần).
- `PERMISSION_INVALIDATION_AUDIT_LOG` (mới) — log đầy đủ ai revoke, quyền gì, khi nào, phục vụ audit an ninh.

```mermaid
erDiagram
    USER ||--o{ USER_PERMISSION : has
    USER ||--o{ PERMISSION_CACHE_ENTRY : "cached per instance"
    USER ||--o{ PERMISSION_INVALIDATION_EVENT : "triggers on"
    PERMISSION_INVALIDATION_EVENT ||--o{ PERMISSION_INVALIDATION_ACK : "acknowledged by"
    PERMISSION_INVALIDATION_EVENT ||--|| PERMISSION_INVALIDATION_AUDIT_LOG : "recorded as"

    USER {
        string id PK
        string name
    }
    USER_PERMISSION {
        string id PK
        string user_id FK
        string permission
        string role
    }
    PERMISSION_CACHE_ENTRY {
        string cache_key PK "user_id + instance_id"
        string instance_id
        string permissions_snapshot
        datetime cached_at
        int ttl_seconds "vd 5, dự phòng"
    }
    PERMISSION_INVALIDATION_EVENT {
        string id PK
        string user_id FK
        string revoked_permission
        string triggered_by_admin_id
        datetime triggered_at
        string status "broadcasting | confirmed | partial"
    }
    PERMISSION_INVALIDATION_ACK {
        string id PK
        string invalidation_event_id FK
        string instance_id
        datetime acknowledged_at
        string status "acked | timeout"
    }
    PERMISSION_INVALIDATION_AUDIT_LOG {
        string id PK
        string invalidation_event_id FK
        string admin_id
        string user_id
        string permission
        datetime triggered_at
        string broadcast_summary
    }
```

---

# Enhance sequence — Admin revoke permission (broadcast + ack + audit)

Đây là **enhance**, cùng flow "Admin revoke permission" đã có ở base nhưng nay thay đổi hoàn toàn theo yêu cầu 1, 2 và 5 của đề bài: broadcast invalidation tới toàn bộ instance qua pub/sub, chờ xác nhận trước khi coi revoke đã có hiệu lực đầy đủ, và log lại toàn bộ cho audit.

```mermaid
sequenceDiagram
    actor Admin
    participant AdminInstance as Service Instance xử lý request admin
    participant DB as USER_PERMISSION store
    participant PubSub as Pub/Sub broadcast
    participant InstanceA as Instance A
    participant InstanceB as Instance B
    participant Audit as PERMISSION_INVALIDATION_AUDIT_LOG store

    Admin->>AdminInstance: Revoke quyền admin của user X
    AdminInstance->>DB: Cập nhật USER_PERMISSION (xoá quyền)
    DB-->>AdminInstance: Cập nhật thành công
    AdminInstance->>PubSub: Publish PERMISSION_INVALIDATION_EVENT (user_id=X, revoked_permission=admin)
    par Broadcast tới toàn bộ instance
        PubSub->>InstanceA: Nhận event, invalidate PERMISSION_CACHE_ENTRY(user_id=X) cục bộ
        InstanceA-->>PubSub: Gửi PERMISSION_INVALIDATION_ACK (status=acked)
    and
        PubSub->>InstanceB: Nhận event, invalidate PERMISSION_CACHE_ENTRY(user_id=X) cục bộ
        InstanceB-->>PubSub: Gửi PERMISSION_INVALIDATION_ACK (status=acked)
    end
    PubSub-->>AdminInstance: Tổng hợp ACK từ toàn bộ instance đã biết
    alt Toàn bộ instance đã ack
        AdminInstance->>AdminInstance: PERMISSION_INVALIDATION_EVENT status=confirmed
        AdminInstance->>Audit: Ghi PERMISSION_INVALIDATION_AUDIT_LOG (admin, user, permission, thời điểm, broadcast_summary=full)
        AdminInstance-->>Admin: "Revoke đã có hiệu lực trên toàn bộ instance"
    else Có instance không ack (mất kết nối tạm thời)
        AdminInstance->>AdminInstance: PERMISSION_INVALIDATION_EVENT status=partial
        AdminInstance->>Audit: Ghi log kèm rõ instance nào không ack
        Note over AdminInstance: Instance chưa ack vẫn được bảo vệ bởi ttl_seconds rất ngắn (xem flow "Authorize request") — cửa sổ rủi ro bị giới hạn tối đa
        AdminInstance-->>Admin: "Revoke có hiệu lực ngay ở đa số instance, 1 instance đang chờ xác nhận"
    end
```

---

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
