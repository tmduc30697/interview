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
